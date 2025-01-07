CDS Declaration Submission Service Design  

**Abstract**

This document defines the functional aspects for submitting import and export declarations - and associated additional messages - into CDS.

This document continues to be updated alongside the agile delivery process of CDS and is therefore subject to change. Where the programme has not yet fully elaborated a requirement in a specific area, this has been identified within the document.

This document supersedes the previously separately issued Imports and Exports Declaration Submission Service Design documents (issued prior to April 2021).

**Original Document Authors:** BW, NA  

**Date Issued:** 11.01.2024

**Status:** Issued

Prepared by: HMRC (CDIO (C&IT) – CDS)

**Contents**

1\. Introduction 3

1.1 Scope 3

1.2 References 3

1.3 Key Updates 4

2\. Scope for Declaration Management 5

2.1 Key Functional Differences from CHIEF (Imports and Exports) 5

2.2 Technical Validation Carried out on all Declarations 9

3\. Imports Declaration Submission 12

3.1 Imports Process Flow 12

3.2 Imports Message Flow 13

3.3 Key Functional Differences from CHIEF for Imports 14

4\. Exports Declaration Submission 26

4.1 Exports Process Flow 26

4.2 Exports Message Flow 28

4.3 Key Functional Differences from CHIEF for Exports 28

5\. Additional Messages 33

5.1 Amendments 33

5.2 Cancellation specific considerations 39

5.3 Non-Inventory Goods Arrival specific considerations 39

5.4 Non-CSP specific considerations 40

6\. Trader Notifications (Imports / Exports) 41

6.1 Types and Definitions 41

6.2 Notification Message Structure 44

6.3 Pointers and Error Codes 69

7\. Interface Definitions 80

7.1 Function Codes and Meta-Data 80

7.2 Data Element Matrix 80

8\. Schemas 80

9\. Example of Export accompanying document (EAD) 81

10\. Document Control 81

#

# Introduction

## Scope

This document defines the system behaviour for the submitting of a declaration, an amendment, an invalidation, or a goods arrival message, and the resulting trader notifications. All processing is largely triggered from the receipt of a declaration message in the WCO format. The document provides an initial overview, and then describes each of the main transactions.

This document covers the submission of Imports and Exports declarations.

Where XML is mentioned in this document, for declaration messages it refers to the UCC/WCO declaration standard used by CDS. Note that external CDS interfaces follow the WCO format, but internal validation will follow the UCC/EUCDM (version 1.1) standard.

This document does not include any definition for how inventory linking will be processed, including goods arrival. This is covered in the specific (Import/Export) Inventory Linking Service Design documents \[3, 8\].

This document does not cover connectivity or the requirements on how to populate the header record, please refer to the API specification \[1\].

# Scope for Declaration Management

For imports and exports, the submitter of the declaration interacts with CDS to manage events such as:

- the **declaration submission.** CDS expects to receive declaration data within the WCO format, defined by the schema and the HMRC Customs Tariff. The schema keeps most elements as optional to allow for capturing a complete declaration, or just partial (e.g. to support simplified import declarations).
- the **request for** **amendment** message is based on the definition of the declaration. The main difference is the change to the functional code of the message. The message structure also uses pointers so CDS can understand which data element needs to be altered within the declaration. For example, a change to the gross mass would require a pointer to the relevant element (one for each ‘layer’ of the message), but the message must also contain the new value. The pointers are defined by the WCO data element tags (found in \[4, 5\]). Some amendments will require approval by customs.
- the **request for invalidation.** Submitted to allow for a trader-initiated cancellation. Some invalidations will require approval by customs.
- receiving the **trader notifications**. As the declaration goes through its lifecycle, certain notifications may be triggered back to the trader. At a bare minimum, a notification of acceptance and clearance can be expected, though others may be sent due to control, liabilities due, etc. A full table is listed below in section Types and Definitions.
- the **goods arrival** message is submitted to instruct CDS that goods are available for inspection following the submission of a pre-lodged declaration.
- For exports specifically;
- manipulating **consolidations** – the management of the relationship between MUCRs, DUCRs, and other MUCRs. Separate transactions are used to associate, or disassociate, two consignment references at a time (a MUCR with another UCR). There is a separate message type for shutting consolidations.

## Key Functional Differences from CHIEF (Imports and Exports)

The key functional differences affecting both Imports and Exports are outlined here. Differences affecting solely imports or exports are covered in sections and respectively.

### Release vs. Clearance

CDS has a difference between the release of the goods and the clearance of the declaration. Clearance signifies that all activities relating to the automatic processing of the declaration have been completed (though there may still need to be some manual finalisation required). The majority of the time the declaration will go straight through to clearance. Release implies that enough processing has taken place to release the goods, but there still may be some activities that require finalisation post-release. The majority of these scenarios are where the debt has been calculated as provisional. The scenarios are:

- The declaration type is a simplified declaration; therefore, the goods can be released, but the declaration cannot be cleared until the missing data has been provided.
- The debt calculation has been calculated as provisional due to either:
  - A quota has been requested. This will cause the customs debt to be automatically re-calculated once CDS received the allocation from TAXUD. Once the calculation has been completed, the declaration is cleared.
  - The measure that the goods have attracted is marked as provisional due to an anti-dumping or a countervailing investigation. The re-calculation is then done when the measure has become finalised, and the declaration is subsequently cleared. See section 6.2.2.11.3 Provisional Anti-Dumping Duty (PDD) for details.
- There is a non-blocking control task raised that is yet to be completed. Once completed, the declaration is then cleared.

For the purposes of Exports, the terms _Release_ and _Clearance_ equate to Permission to Progress (P2P).


### Tax Point / Legal Acceptance

For CDS, the legal acceptance of the declaration is defined through the DMSACC notification. This notification will include the datetime stamp as to when that specific moment was. The legal acceptance also defines the tax point as to when the debt was incurred.

### MRN as the Unique Declaration ID

CDS will not have a separate entry number. The unique ID for the declaration will be the MRN. This MRN will be retained throughout the declaration lifecycle, regardless of the number of amendments made. The ID will be returned to the submitter through the initial DMS notification to the declaration submission. This can be a response notifying of either acceptance, registration, or rejection.

The structure of the MRN is as follows:

| **Content** | **Format** | **Example** |
| --- | --- | --- |
| Last 2 digits of system date | n2  | “17” |
| Identification of the Member State where the MRN is generated | a2  | “GB” |
| Unique identifier per year and country Optionally the 7th position in this field (i.e. the 11th position in the full number) is a configurable alphanumeric character that is determined via a system parameter. This character can be used as indicator for the kind of process for which the MRN is issued in order to make it unique over multiple systems. In case the system parameter is missing or empty then this option is not used. | an13 | “9876AB8890123” |
| Check digit according to ISO 6346 | an1 | “5” |

### FEC no longer blocking

FEC checks will be run during the validation process. Any issues identified will be sent back to the submitter via the DMSACC and DMSRCV notifications as a ‘CredibilityValidationResult’, however the declaration will continue along the declaration journey. The submitter will have until the end of the 10-minute dwell time to amend the declaration if necessary, else the declaration will be processed as submitted.

It is likely that the validation results in the DMSRCV and DMSACC will be identical. The only possible exception may be pre-lodged declarations where the parameters used for the checks may have changed between lodging and arrival.

### Requested supporting documents can now be uploaded rather than emailed or faxed

In addition to the current route where a trader emails his documents to NCH for manual upload by NCH workers into CDS, a digital route will be provided to the traders to submit their documents in the form of the CDS Digital Secure File Upload Service. The Service will be delivered as an API to be invoked by 3rd Party Software that requires the ability to provide this functionality to traders, alongside other CDS UIs (including HMRC’s own external Secure File Upload UI for traders). An authenticated trader may use the service to submit files containing their documents required to support the declarations made by them. In some cases, a trader may have also received a DMSDOC notification triggered by the declaration risking process to submit their documents for a declaration that they have submitted. (Refer to 2.2.4 Payload Size for maximum file size)

The API will accept requests from the trader’s software to upload a number of files for a given declaration ID (MRN) and respond with a location (URL) to upload each file. The trader will then upload each file into an inbound file bucket. The API will return a unique file reference code to the software for each file successfully submitted by the trader via a notification. This file reference code can be used in future correspondence with HMRC. HMRC Caseworkers will be notified when each document uploaded by the trader is received and also when all files uploaded together as a group for a declaration have been received. When all files have been received, HMRC Caseworkers will review the documents they contain against the case for the declaration ID provided in the original API call. If the HMRC Caseworker finds any issues, they will notify the trader to upload their documents again. In such cases, the HMRC caseworker will also reset any SLAs defined around this process. The files uploaded by the trader will be scanned for malware and then stored in HMRC’s document repository system where they will be retained in accordance with HMRC policies (6 + current year).

### All declarations come through one API and traders need to be individually registered

CDS supports declaration submission from traders using their own/on-premises software or compatible Software-as-a-Service (SaaS) offerings.

Similar to other services on HMRC’s MDTP platform, standard patterns are available for SaaS providers to facilitate the authentication of their clients to enable them to submit declarations to CDS, that can be used depending on their service operating model (e.g. acting as an agent or facilitating direct trader submission).

SaaS based solutions will need to register one call-back URL. For separate software installed at client's premises then each client must register the software on Dev Hub using the call-back option that the client selects (push or pull).

More information can be found in the CDS Vol 3, Import Tariff \[6\], at <https://developer.service.hmrc.gov.uk/api-documentation/docs/authorisation/user-restricted-endpoints>, and on the Developer Hub.

### Declarations must be submitted individually - batch submission is not available

Declarants must submit declarations individually to the CDS service, following the process as described in this document and the _End to End Sequence Diagrams_ \[2\].

It is not possible to submit declarations to CDS in a batch.

### Handling of duplicate submissions

CDS will identify duplicates through the uniqueness of the submitterReference field (EUCDM 2/5 LRN) against all accepted and registered declarations for the submitter. References against rejected declarations can be re-used. Where a duplicate is identified the declaration will be rejected with a specific validation result type. See section Error Codes and the Codelists \[4\] for the full list of error codes.

### Declarations under control – Communication with HMRC Caseworker

Where a declaration is placed under control, separate to a DMSDOC/DMSCTL notification, the HMRC caseworker can send a message with queries/instructions to the declarant where this is necessary.

When a message is sent by the caseworker, a notification is sent by email to the registered email address of the declarant (this is the email address as supplied by the declarant when initially registering for CDS).  
For security reasons, the email will not contain the caseworker comments, instead this will advise of a secure message having been sent and is awaiting action.

In addition to the email notification, a new CDS notification type – Query Notification Message (DMSQRY) will also be sent to alert the user through their software that the declaration is under query and requires their action. It is not possible to respond to this message, like the email, it is just an alert to access the query online.

To access the message content, the declarant is required to login to the Secure File Upload service, using the related Government Gateway credentials. Here, the declarant can read the message and determine the action(s) necessary to resolve the query. For the declaration to progress it is necessary for the declarant to respond explicitly to the query (with a secure message) in addition to taking any requested action(s) such as uploading documents.

Depending on the nature of the query, it is possible the above steps may repeat. When the caseworker has determined the required action(s) have been satisfied, the declaration will continue processing.

**Note**,

1. The Secure File Upload service is the only location where the caseworker message can be read – this message does not feature in the DMSDOC/DMSCTL message nor is visible in a Declaration Information Service query.

### Amendments after Rejection

Chief allows amendments to be made to declarations after rejection. For CDS, DMSREJ is an end state for a declaration. After a DMSREJ is received the only option is to submit a new declaration with any errors listed in the DMSREJ corrected.

Note that the rejection of a declaration is different to the rejection of an amendment, GPN or cancellation. A declaration is only rejected if a DMSREJ is received with the same conversationId as the declaration. If a DMSREJ is sent for an amendment, GPN or cancellation request, the original declaration is still valid and remains in the system unaltered.

The declaration will not, however, progress normally until the rejected amendment is resolved either by submitting an amendment that addresses the reason(s) for rejection, or by submitting a nil-amendment that returns the declaration to the original position (i.e., the attempt to amend is withdrawn).

Refer to 5.2 on how to progress when a declaration is rejected or invalidated.

Note: Conversation ID is an identifier which is returned as a header with the synchronous HTTP response. The Conversation ID is unique to (but shared by) a submitted message and any of its corresponding notifications. For example, a CDS declaration which has been submitted successfully might trigger an HTTP 202 (Accepted) response, the header of that 202 will contain a Conversation ID, and the DMSACC, DMSTAX, DMSTAX, and DMSCLE which subsequently get created as part of the declaration processing will all share that same Conversation ID.

## Technical Validation Carried out on all Declarations

Import and Export Declarations as well as Additional Messages go through three stages of validation before being processed.

### XSD Schema Validation

The first stage is XSD schema validation. This ensures that the received document matches the schema defined for that API. If your message fails validation against the XSD, a HTTP 400 response status code will be returned. The payload will indicate the elements failing validation. Note that a DMSREJ will not be sent as the validation checking is performed at the API gateway. Documents failing schema validation are rejected before reaching CDS for processing. The XSDs are available on the Developer Hub.

Example response body in the case of schema validation error:

&lt;?xml version='1.0' encoding='UTF-8'?&gt;

&lt;errorResponse&gt;

&lt;code&gt;BAD_REQUEST&lt;/code&gt;

&lt;message&gt;Payload is not valid according to schema&lt;/message&gt;

&lt;errors&gt;

&lt;error&gt;

&lt;code&gt;xml_validation_error&lt;/code&gt;

&lt;message&gt;cvc-datatype-valid.1.2.1: 'X' is not a valid value for 'decimal'.&lt;/message&gt;

&lt;/error&gt;

&lt;error&gt;

&lt;code&gt;xml_validation_error&lt;/code&gt;

&lt;message&gt;cvc-type.3.1.3: The value 'X' of element 'SequenceNumeric' is not valid.&lt;/message&gt;

&lt;/error&gt;

&lt;/errors&gt;

&lt;/errorResponse&gt;

Further details on XML validation error formats can be found in the online Developer guide Reference section and as part of the online documentation for the individual APIs.

### Web Application Firewall (WAF) Validation

The entire document is then checked to ensure it does not contain certain keywords known to damage or create unexpected behaviour in Web Applications. The rules used are open source and managed by the OWASP foundation.

Currently, a HTTP 500 response code will be sent with a generic message body if your document fails this stage. As with XSD schema checks above, no DMSREJ will be sent as the document is rejected before it reaches the processing stage.

Response body in the case of a WAF validation error:

&lt;?xml version='1.0' encoding='UTF-8'?&gt;

&lt;errorResponse&gt;

&lt;code&gt;INTERNAL_SERVER_ERROR&lt;/code&gt;

&lt;message&gt;Internal server error&lt;/message&gt;

&lt;/errorResponse&gt;

A HTTP 403 response code will be returned with a generic message body if your document fails this stage. As with XSD schema checks above, no DMSREJ will be sent as the document is rejected before it reaches the processing stage.

Response body in the case of a WAF validation error:

&lt;?xml version="1.0" encoding="UTF-8"?&gt;

&lt;errorResponse&gt;

&nbsp;  &lt;code&gt;PAYLOAD_FORBIDDEN&lt;/code&gt;

&nbsp;  &lt;message&gt;A firewall rejected the request”&lt;/message&gt;

&lt;/errorResponse&gt;

Note that the WAF validation ruleset are expressed as regular expressions (patterns) rather than explicit lists of keywords, so it is not possible to provide a list of potential keywords that will cause a document to fail this validation phase. However, we can provide examples of word and character combinations that have been reported as causing rejections. These are available on the Developer Hub, along with suggestions to avoid the error.

### Business Rules Validation

The third and final stage is validation against business rules. The Business Rules validate that the declaration is UCC compliant and adheres to the Paper Tariff, in addition to the validity & applicability of licenses, parties, and accounts.

A HTTP 200 response code will be sent at this stage as, strictly speaking, the document has reached the processing stage. However, if the document fails any of the business rules, an asynchronous DMSREJ will be sent, containing the error codes and pointers to the fields failing validation.

### Payload Size

The maximum payload size for any message is 10Mb. Acceptable file formats are: jpeg, pdf, MS Word(.doc, .docx), MS Excel(.xls, .xlsx) or png.

Payloads larger than this will receive a HTTP 413 response code with HTML body.

# Imports Declaration Submission

## Imports Process Flow

### Introduction

The below process describes the CDS process for managing import declarations. This is the main over-arching ‘process controller’ which manages when the other sub-processes are called. It is depicted here to provide a view as to what happens when in the declaration lifecycle. The message flows in \[2\] provide more detail on the external interactions.



| **ID** | **Sub-Process** | **Description** |
| --- | --- | --- |
| VAL | Declaration Validation and Indicative duty calculation | The first sub-process consists of validating the semantic correctness and completeness of the declaration, primarily based on the declaration type and the procedure category. The declaration may be rejected at certain points in the process if the negative result is identified, without completing the full validation process. A notification will be submitted to the trader to inform them of the Customs decision on the declaration (acceptance, registration or rejection).<br><br>An indicative duty calculation will be provided for valid declarations (of type A, B, D, E, J or K) at this stage.<br><br>If a pre-lodged declaration is registered, but not arrived within 30 days, it will be rejected. |
| RISK | Risk | The declaration data is passed to risk assessment which may trigger relevant control actions to be performed. |
| QUO | Quota Request and Allocation | Where there is a new, or changed, quota request on the declaration, there is a formal request to TAXUD for the amount required. When an allocation is recorded (up to 2 days later), the tax calculation is performed again, and any delta is handled. |
| CTRL | Initiate and Perform Control | If the risk assessment identifies any potential actions to take place, this sub-process manages the tasks to be completed. A documentary control notification may be sent to the submitter at this point, and if the goods are on hand, a physical control notification.<br><br>Documentary controls will remain open for a maximum of 1024 days (3 years) before being referred to a caseworker for manual review. Physical/ALVS controls will remain open for a maximum of 180 days.<br><br>If the declaration is pre-lodged, the flow effectively stops here. No customs position is taken on a pre-lodged declaration. |
| POS | Determine Customs Position | Following receipt of control results, CDS will take a customs position on the declaration. The decision is made to either release the goods, clear the declaration, or invalidate.<br><br>Within this process a 10-minute dwell time starts to allow traders to submit an amendment if any corrections need to be made. Note that after this point in the process, no further amendments will be accepted. |
| DEBT.CALC | Debt Calculation | If the declaration involved customs duties, this process will calculate the final duty amount, including any relief or suspension. A trader notification will be sent at this to either inform of the debt reserved against a deferment account, or as a payment instruction if the method of payment is an immediate payment type.<br><br>Where a goods item has attracted provisional anti-dumping/countervailing duty, and the measure is subsequently finalised, the declaration process will restart from this point to calculate the final customs debt, and to allow the declaration to proceed to customs debt readjustment and clearance. |
| DEBT.PAY | Duty Coverage | Where duties are calculated, this process handles the coverage of those customs debts. This can be one of, or a combination of, immediate or deferred payment. These two sub-processes cover all MOP types.<br><br>Full payment must be made within 30 days. On day 31 the declaration will be passed to a caseworker for manual resolution of the debt. |
| CLR | Clear Declaration | The final process handles the clearance or release of goods. The distinction between the two is described above in section Release vs. Clearance. |

## Imports Message Flow

The end to end sequence diagrams can be found in CDS End to End Sequence Diagrams \[2\] and are therefore not replicated here.

The relevant sequence diagrams that relate to Import Declaration Submission are:

- Submit Pre-Lodged Declaration
- Submit Goods Arrival
- Submit Arrived Declaration
- Submit Supplementary Declaration
- Submit Amendment Request
- Submit Invalidation Request

## Key Functional Differences from CHIEF for Imports

This section outlines the key differences from CHIEF affecting Import declarations only, for key differences affecting both Imports and Exports, please refer to section

### Declaration Amendment Restrictions

Currently, CDS will not allow an amendment after a customs position has been taken on the declaration. Section Flow Diagram and Sub-Process Description describes where this is within the over-arching process. Traders will still have a 10-minute dwell time to amend the declaration, however, it will currently be at the point just before the custom position, rather than after the final duty calculation.

### Guarantees and Immediate Methods of Payment

There are two generic processes for handling coverage of the customs duties liable that are triggered dependent on the MOP code: Immediate and Guarantee. Please note that this is in regard to MOP categories and how they are processed, rather than actual debt (payment due) and potential debt (to be guaranteed).

An immediate payment is defined as a method of payment which requires a transfer of funds at that point to HMRC, e.g. bank transfer, debit card payment etc. At the point where payment is required, a DMSTAX notification will be used as a payment instruction and will contain a specific reference (prefixed CDSI…) to use when the payment is made. The goods will not be released until HMRC receive a confirmation from the bank that the payment has covered the liability. The speed at which HMRC are notified of the payment will be dependent on the method of payment used. Banks have an SLA of 2 hours to notify of Faster Payments, whereas others are often constrained by banking hours, bank holidays, and have differing SLAs.

The guarantee process is one where there is a balance of funds to reserve the duties against, e.g. deferment accounts, guarantee accounts, which then allow the declaration to be released/cleared if the funds are available. If the funds available are not sufficient to cover the debt, then a notification (DMSCPI) will be raised to the submitter of the declaration, informing them of the need to pay down some debt or increase the guarantee threshold. If the payer is a different individual to the submitter of the declaration, then it is the submitter’s responsibility to inform the owner of the account to update the balance.

Please note that CDS does not currently support an individual/differing method of payment for each tax line on the declaration - CDS will only take payments via one type of MoP on a single declaration, but where security is required this can be combined with a guarantee account in certain cases (DAN1 and GAN; CAN and GAN – see below).

The combinations currently supported on a single declaration are:

DAN1 (E/R)  
DAN1 and DAN2 (E/R)  
DAN1 and GAN (E/R + S/T/U/V)  
CAN and GAN (N/P + S/T/U/V)  
CAN (N/P)  
GAN (S/T/U/V)  
Immediate/Cash (A/M or no MoP declared anywhere else on the declaration\*

PVA is compatible with the above where payment is required, however DAN2 is not compatible with use of PVA on the same declaration.

\*Note that if MoP A is entered anywhere in the declaration, CDS will default to immediate payment on the entire declaration, unless otherwise rejected by business rules.

It should be noted that if the intention is to combine payment and security on the declaration (for example, DAN1+GAN etc), but the requirements are only completed for one specific MoP and not rejected by business rules (for example, DAN1 completion requirements met, GAN requirements not met), CDS will process the payment and security from the correctly completed MoP (DAN1 etc).

If a MoP is omitted against one tax line on the dec, the overall MoP will apply to the declaration - for example, if the declaration is completed according to deferment, and one goods item does not have a MoP entered against that tax line, it will default to deferment, unless business rules will otherwise reject the declaration (if document code C506 is present against a goods item, rules will expect a deferment MoP and reject otherwise).

Note: Where no method of payment has been entered in the declaration, and CDS identifies that there are charges due, CDS will default to immediate cash payments for the declaration. CDS does not issue a DMSCPI message for this, as this scenario does not cover an actual account with an insufficient balance. In this scenario (and also where immediate cash payments are explicitly declared), CDS provides the payment reference within the DMSTAX notification

### Processing of Quotas

Whereas CHIEF would handle quota allocation and securities off-system following the clearance of the entry, CDS processes them during the declaration lifecycle. This means that a quota allocation request is captured at the point of declaration acceptance.

Amendments made by the trader to the declaration within the dwell time (such as a trader identifying a completion error following the indicative duty calculation being provided \[see Indicative Duty Calculation (Pre-Dwell-Time)\]) will amend the quota allocation request as appropriate.

Once the dwell time has finished, a duty calculation is performed using the preferential rate, and with a security amount taken (value is the difference between quota & non-quota rate) if the quota is in a critical status.

When the provisional customs debt has been covered, the declaration will be granted “release” as opposed to clearance (see section Release vs. Clearance for further information).

Once the quota allocation has been finalised, CDS recalculates a final customs debt reflecting this allocation, and readjusts the owed duties accordingly in a second DMSTAX trader notification, including the refunding of any security taken as necessary. As long as there are no outstanding pre-finalisation activities after this, the declaration can go on to be granted clearance.

#### Processing of Partial Quota Allocations

Where a trader submits a claim to quota that exceeds the available amount remaining in the quota (for example, a quota request is submitted on the last date that a critical quota is available) the trader may receive a partial quota allocation. Part of their quota request will be granted the “in-quota” duty rate and the rest will receive the “out of quota” (non-preferential) rate. The following section highlights the current structure of the DMSTAX response in this scenario.

Given an example Declaration Submission of 3 Goods Items, 2 of which are requesting a Quota and have a Duty Regime Code of 320.

| **Goods Item** | **Commodity** | **Duty Regime Code** | **Quota Status** | **Quota ID** | **Goods Value** |
| --- | --- | --- | --- | --- | --- |
| 1   | 20094192 20 | 320 | Critical | 051867 | 1400 |
| 2   | 73045999 00 | 100 | N\\A | N/A | 1000 |
| 3   | 20094192 20 | 320 | Critical | 051867 | 600 |

The 3 goods items are shown in the following stripped-down XML snippets:

[&lt;dec:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:SequenceNumeric&gt;1&lt;/dec:SequenceNumeric&gt;

[&lt;dec:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)

[&lt;dec:Classification&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:ID&gt;20094192&lt;/dec:ID&gt;&lt;/dec:Classification&gt;

[&lt;dec:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:DutyRegimeCode&gt;320&lt;/dec:DutyRegimeCode&gt;

&lt;dec:QuotaOrderID&gt;051867&lt;/dec:QuotaOrderID&gt;&lt;/dec:DutyTaxFee&gt;

[&lt;dec:InvoiceLine&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)

&lt;dec:ItemChargeAmount currencyID="GBP"&gt;1400.00&lt;/dec:ItemChargeAmount&gt;

&lt;/dec:InvoiceLine&gt;

&lt;/dec:Commodity&gt;

&lt;/dec:GovernmentAgencyGoodsItem&gt;

[&lt;dec:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:SequenceNumeric&gt;2&lt;/dec:SequenceNumeric&gt;

[&lt;dec:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)

[&lt;dec:Classification&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:ID&gt;73045999&lt;/dec:ID&gt;&lt;/dec:Classification&gt;

[&lt;dec:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:DutyRegimeCode&gt;100&lt;/dec:DutyRegimeCode&gt;&lt;/dec:DutyTaxFee&gt;

[&lt;dec:InvoiceLine&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)

&lt;dec:ItemChargeAmount currencyID="GBP"&gt;1000.00&lt;/dec:ItemChargeAmount&gt;

&lt;/dec:InvoiceLine&gt;

&lt;/dec:Commodity&gt;

&lt;/dec:GovernmentAgencyGoodsItem&gt;

[&lt;dec:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:SequenceNumeric&gt;3&lt;/dec:SequenceNumeric&gt;

[&lt;dec:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)

[&lt;dec:Classification&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:ID&gt;20094192&lt;/dec:ID&gt;&lt;/dec:Classification&gt;

[&lt;dec:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)&lt;dec:DutyRegimeCode&gt;320&lt;/dec:DutyRegimeCode&gt;

&lt;dec:QuotaOrderID&gt;051867&lt;/dec:QuotaOrderID&gt;&lt;/dec:DutyTaxFee&gt;

[&lt;dec:InvoiceLine&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\declaration%20%282%29.xml)

&lt;dec:ItemChargeAmount currencyID="GBP"&gt;600.00&lt;/dec:ItemChargeAmount&gt;

&lt;/dec:InvoiceLine&gt;

&lt;/dec:Commodity&gt;

&lt;/dec:GovernmentAgencyGoodsItem&gt;

We next explore current DMSTAX quota processing behaviour and a proposed way to overcome its shortcomings in the future.

##### DMSTAX Current Quota Processing

After a partial quota allocation in response to a request for a critical quota, CDS responds in the DMSTAX with separate calculations for the In Quota and Out of Quota elements of the allocation. It does this by splitting the submitted GoodsItem in 2 deriving a new GoodsItem for the Out of Quota element of each response.

CDS currently processes the In-Quota elements (preference code 320) against the original GoodsItem(1, 3 in the example), creating 2 additional Out of Quota GoodsItem 4,5 (split out from goods items 1, 3) therefore returning 5 GoodsItem in DMSTAX.

The original GoodsItem with a request to quota are allocated a duty calculation showing the in-quota preference code of ‘320’ and the new GoodsItem that are out of quota are allocated a duty calculation against the ‘non-preferential’ rate with a preference code of ‘100’ being shown. In addition, two VAT tax amounts are returned, one amount against the GoodsItem allocated quota and a second amount against the new GoodsItem that did not benefit from quota.

These additional GoodsItem are for internal CDS processing and should not be exposed to the trade as additional GoodsItem, as Trader software is unable to associate the extra GoodsItem with the original declaration submission. This behaviour will be remediated in a future release with a timeline yet to be agreed.

This section will be updated in a subsequent issue to provide the DMSTAX response that will be provided where goods liable to excise tax types receive a partial quota allocation and the DMSTAX response provided for a zero-quota allocation.

| **Goods Item** | **Commodity** | **Duty Regime Code** | **Quota Status** | **Quota ID** | **Tax Code** | **Details** | **AdValorem Tax Base Amount** | **Tax Rate Numeric** | **Tax Assessed Amount** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1   | 20094192 20 | 320 | In Quota | 051867 | A00 | Duty | 280 | 5.8 | 16.24 |
|     |     |     |     |     | B00 | VAT | 296.24 | 20.0 | 59.24 |
| 2   | 73045999 00 | 100 | Out of Quota | N/A | A00 | Duty | 1000 | 0   | 0   |
|     |     |     |     |     | B00 | VAT | 1000 | 20  | 200 |
| 3   | 20094192 20 | 320 | In Quota | 051867 | A00 | Duty | 120 | 5.8 | 6.96 |
|     |     |     |     |     | B00 | Duty | 126.96 | 20.0 | 25.39 |
| 4   | 20094192 20 | 100 | Out of Quota | 051867 | A00 | Duty | 1120 | 14.0 | 156.80 |
|     |     |     |     |     | B00 | VAT | 1276.80 | 20.0 | 255.36 |
| 5   | 20094192 20 | 100 | Out of Quota | 051867 | A00 | Duty | 480 | 14.0 | 67.20 |
|     |     |     |     |     | B00 | Duty | 547.20 | 20.0 | 109.44 |

The 3 GoodsItem expected in DMSTAX response corresponding to the 3 GoodsItem in Declaration submission is shown in the following stripped-down XML snippets.

Declaration Submission GoodsItem1 Goods Value 1400 = DMSTAX response GoodsItem1 InQuota Duty Tax Base Amount 280 + DMSTAX response GoodsItem 4 Out of Quota Duty Tax Base Amount 1120

GoodsItem 1

[&lt;\_2_1:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)&lt;\_2_1:SequenceNumeric&gt;1&lt;/\_2_1:SequenceNumeric&gt;

[&lt;\_2_1:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;280.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;320&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;5.80&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;16.24&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;16.24&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;296.24&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;320&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;59.24&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;59.24&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

&lt;/\_2_1:Commodity&gt;

&lt;/\_2_1:GovernmentAgencyGoodsItem&gt;

GoodsItem 2

[&lt;\_2_1:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)&lt;\_2_1:SequenceNumeric&gt;2&lt;/\_2_1:SequenceNumeric&gt;

[&lt;\_2_1:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;1000.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;0.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;1000.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;200.00&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;200.00&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

&lt;/\_2_1:Commodity&gt;

&lt;/\_2_1:GovernmentAgencyGoodsItem&gt;

Declaration Submission GoodsItem3 Goods Value 600 = DMSTAX response GoodsItem3 InQuota Duty Tax Base Amount 120 + DMSTAX response GoodsItem 5 Out of Quota Duty Tax Base Amount 480

GoodsItem 3

[&lt;\_2_1:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)&lt;\_2_1:SequenceNumeric&gt;3&lt;/\_2_1:SequenceNumeric&gt;

[&lt;\_2_1:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;120.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;320&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;5.80&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;6.96&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;6.96&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;126.96&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;320&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;25.39&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;25.39&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

&lt;/\_2_1:Commodity&gt;

&lt;/\_2_1:GovernmentAgencyGoodsItem&gt;

The structure of the 2 additional **unexpected** GoodsItem in DMSTAX response resulting from Out Of Quota processing internal to CDS is shown in the following stripped-down XML snippets (This behaviour will be remediated in a future release and the next section shows the proposed approach).

GoodsItem 4

[&lt;\_2_1:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)&lt;\_2_1:SequenceNumeric&gt;4&lt;/\_2_1:SequenceNumeric&gt;

[&lt;\_2_1:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;1120.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;14.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;156.80&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;156.80&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;1276.80&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;255.36&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;255.36&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

&lt;/\_2_1:Commodity&gt;

&lt;/\_2_1:GovernmentAgencyGoodsItem&gt;

GoodsItem5

[&lt;\_2_1:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)&lt;\_2_1:SequenceNumeric&gt;5&lt;/\_2_1:SequenceNumeric&gt;

[&lt;\_2_1:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;480.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;14.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;67.20&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;67.20&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;547.20&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;109.44&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;109.44&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

&lt;/\_2_1:Commodity&gt;

&lt;/\_2_1:GovernmentAgencyGoodsItem&gt;

##### DMSTAX Proposed Quota Processing

The following proposed structure shows how the split calculations can be set within the same GoodsItem

In Quota Preference Code 320

- A00 Duty Calculation
- B00 VAT Calculation

Out of Quota Preference Code 100

- A00 Duty Calculation
- B00 VAT Calculation

CDS will process and return In Quota and Out of Quota calculations within the same originally submitted GoodsItem sequence number in DMSTAX

| **Goods Item** | **Commodity** | **Duty Regime Code** | **Quota Status** | **Quota ID** | **Tax Code** | **Details** | **AdValoreum Tax Base Amount** | **Tax Rate Numeric** | **Tax Assessed Amount** |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1   | 20094192 20 | 320 | In Quota | 051867 | A00 | Duty | 280 | 5.8 | 16.24 |
|     |     |     |     |     | B00 | VAT | 296.24 | 20.0 | 59.24 |
|     |     | 100 | Out of Quota | 051867 | A00 | Duty | 1120 | 14.0 | 156.80 |
|     |     |     |     |     | B00 | VAT | 1276.80 | 20.0 | 255.36 |
| 2   | 73045999 00 | 100 | Out of Quota | N/A | A00 | Duty | 1000 | 0   | 0   |
|     |     |     |     |     | B00 | VAT | 1000 | 20  | 200 |
| 3   | 20094192 20 | 320 | In Quota | 051867 | A00 | Duty | 120 | 5.8 | 6.96 |
|     |     |     |     |     | B00 | Duty | 126.96 | 20.0 | 25.39 |
|     |     | 100 | Out of Quota | 051867 | A00 | Duty | 480 | 14.0 | 67.20 |
|     |     |     |     |     | B00 | Duty | 547.20 | 20.0 | 109.44 |

The following stripped-down XML snippets show Out of Quota calculations within same In Quota GoodsItem.

Out of Quote Preference Code 100

- A00 Duty Calculation
- B00 VAT Calculation

GoodsItem 1

[&lt;\_2_1:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)&lt;\_2_1:SequenceNumeric&gt;1&lt;/\_2_1:SequenceNumeric&gt;

[&lt;\_2_1:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;280.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;320&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;5.80&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;16.24&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;16.24&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;296.24&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;320&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;59.24&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;59.24&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;1120.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;14.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;156.80&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;156.80&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;1276.80&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;255.36&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;255.36&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

&lt;/\_2_1:Commodity&gt;

&lt;/\_2_1:GovernmentAgencyGoodsItem&gt;

GoodsItem 2

[&lt;\_2_1:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)&lt;\_2_1:SequenceNumeric&gt;2&lt;/\_2_1:SequenceNumeric&gt;

[&lt;\_2_1:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;1000.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;0.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;1000.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;200.00&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;200.00&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

&lt;/\_2_1:Commodity&gt;

&lt;/\_2_1:GovernmentAgencyGoodsItem&gt;

GoodsItem 3

[&lt;\_2_1:GovernmentAgencyGoodsItem&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)&lt;\_2_1:SequenceNumeric&gt;3&lt;/\_2_1:SequenceNumeric&gt;

[&lt;\_2_1:Commodity&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;120.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;320&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;5.80&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;6.96&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;6.96&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;126.96&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;320&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;25.39&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;25.39&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;480.00&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;14.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;A00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;67.20&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;67.20&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;2_1:DutyTaxFee&gt;

[&lt;\_2_1:DutyTaxFee&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:AdValoremTaxBaseAmount currencyID="GBP"&gt;547.20&lt;/\_2_1:AdValoremTaxBaseAmount&gt;

&lt;\_2_1:DeductAmount currencyID="GBP"&gt;0.00&lt;/\_2_1:DeductAmount&gt;

&lt;\_2_1:DutyRegimeCode&gt;100&lt;/\_2_1:DutyRegimeCode&gt;

&lt;\_2_1:TaxRateNumeric&gt;20.00&lt;/\_2_1:TaxRateNumeric&gt;

&lt;\_2_1:TypeCode&gt;B00&lt;/\_2_1:TypeCode&gt;

[&lt;\_2_1:Payment&gt;](file:///C:\Users\7839936\OneDrive%20-%20HM%20Revenue%20&%20Customs\DMSTAX\Quotas\dmstax%20%284%29.xml)

&lt;\_2_1:TaxAssessedAmount currencyID="GBP"&gt;109.44&lt;/\_2_1:TaxAssessedAmount&gt;

&lt;\_2_1:PaymentAmount currencyID="GBP"&gt;109.44&lt;/\_2_1:PaymentAmount&gt;

&lt;/\_2_1:Payment&gt;

&lt;/\_2_1:DutyTaxFee&gt;

&lt;/\_2_1:Commodity&gt;

&lt;/\_2_1:GovernmentAgencyGoodsItem&gt;

### Specifying multiple methods of payment

#### Specifying one DAN and one GAN

Where the provided procedure code permits, in CDS it is possible to provide a Deferment Account and a Guarantee Account together for use as methods of payment on a Declaration.

Where this is specified on a declaration;

- all outright amounts are reserved against the deferment account
- all security amounts are reserved against the guarantee account.

Both accounts provided must be authorised for use by the trader at the point of duty calculation.  

#### Specifying two DANs

Beyond the scenario described in Specifying one DAN and one GAN, CDS does not offer the ability for a trader to be able to specify the method of payment that will cover specific duty types. For example, if an immediate payment method, such as cash, is specified within the declaration, this will cover all debt associated with the declaration.

There are certain limited scenarios however where charges can be split between multiple accounts: for instance, by declaring a 2nd DAN, this allows VAT to be charged to that separate account. The system will carry this out automatically, and there is no expectation that multiple instances of data element 4/8 need to be declared to achieve this. Where this is applicable, this will be described in the CDS Tariff, but these scenarios are fixed, and the trader will not have the capability to pick and choose how each tax type is paid for.

The exception to this statement is where the trader has performed a manual tax calculation and is declaring the tax amounts to be covered. For example, if paying by deferment, and depending on whether a given amount for a given tax types needs to be paid or secured, the applicable MOP code of ‘E’ or ‘R’ will need to be used accordingly.

### Indicative Duty Calculation (Pre-Dwell-Time)

Indicative duty calculations provide traders an opportunity to spot declaration completion errors, giving chance for traders to amend (or cancel) declarations as appropriate before the dwell time expires.

CDS will calculate indicative duty calculations in the following scenarios;

1\. On initial receipt of a valid declaration of type A, B, D, E, J, or K.

2\. On receipt of a valid goods arrival notification (for pre-lodged declarations).

3\. On receipt of a valid amendment to a declaration type A, B, D, E, J, or K within the dwell time and the changes are accepted.

The indicative duty calculation is sent as a DMSTAX notification, following the same message format as a finalised calculation - but without the inclusion of payment details.

- For declarations of type A, B, D, or E, where a trader has overridden the duty calculation for a goods item in DE 2/2, this will be used as the indicative calculation for that goods item.
- For declarations of type J or K, where duty calculation has been manually provided by the trader, this amount is used as the indicative calculation for that goods item.
- The indicative calculation will apply Relief/Suspension of duties (as appropriate).

### Customs Freight Simplified Procedure (CFSP) – Submitting Final Supplementary Declarations

Customs Freight Simplified Procedure (CFSP) traders can submit Final Supplementary Declarations (FSD). These inform HMRC how many Supplementary Declarations have been submitted and indicate how many are due to be submitted (expected) - to meet the authorisation requirements of the CFSP.

The FSD (Type Q) declaration submission process is as per other declaration types, however with some exceptions such as Front-End Checks (FEC) are not performed, and Duty/Tax calculations will not take place.

FSD’s are amendable by the trader during the Dwell Time, as per other declaration types.

HMRC has produced specific documentation on FSD’s – please refer to the _Final Supplementary Declaration Solution Design_ \[9\]. This details the specific procedure codes and other D.E. values required in order to submit a valid FSD.

# Exports Declaration Submission

## Exports Process Flow

### Introduction

The below process describes the CDS process for managing export declarations. This is the main over-arching ‘process controller’ which manages when the other sub-processes are called. It is depicted here to provide a view as to what happens when in the declaration lifecycle. The message flows in \[2\] provide more detail on the external interactions.

The process can be run multiple times for exports due to the ability to re-arrive at multiple ports before departure.

### Flow Diagram and Sub-Process Description

| **ID** | **Sub-Process** | **Description** |
| --- | --- | --- |
| VAL | Declaration Validation | The first sub-process consists of validating the semantic correctness and completeness of the declaration, primarily based on the declaration type and the procedure category. The declaration may be rejected at certain points in the process if a negative result is identified, without completing the full validation process. A notification will be submitted to the trader to inform them of the Customs decision on the declaration (acceptance, registration or rejection).<br><br>When a declaration is re-arrived, the declaration is revalidated - based largely on the new location of the goods.<br><br>If a pre-lodged declaration is registered, but not arrived within 30 days, it will be rejected. |
| RISK | Risk | The declaration data is passed to risk assessment which may trigger relevant control actions to be performed.<br><br>CSPs will be notified at this point of the control routing, if an EAA or EAL transaction has been submitted. |
| CTRL | Initiate and Perform Control | If the risk assessment identifies any potential actions to take place, this sub-process manages the tasks to be completed. A documentary control notification may be sent to the submitter at this point, and if the goods are on hand, a physical control notification.<br><br>Documentary controls will remain open for a maximum of 1024 days (3 years) before being referred to a caseworker for manual review. Physical/ALVS controls will remain open for a maximum of 180 days.<br><br>If the declaration is pre-lodged, the flow effectively stops here. No customs position is taken on a pre-lodged declaration. |
| POS | Determine Customs Position | Following receipt of control results, CDS will take a customs position on the declaration. The decision is made to either release the goods, clear the declaration, or invalidate. |
| CLR | Clear Declaration | The final process handles the clearance or release of goods. The trader and CSP is notified of the customs decision at this point. |
| MOR | Monitor After Release | Where the declaration is an export type and an exit message is anticipated (i.e. ‘TRANS’ is not declared), this process handles the responses from the Office of Exit and any timers by which an exit confirmation is required by. A reminder is sent to the trader and the Office of Exit for exit confirmation after90 days. The declaration is invalidated after 150 days, if no exit message is received.<br><br>Once an exit message is received, no amendment can be submitted by the trader as the export movement is effectively closed. |
| VAL.MESS | Validate Additional Message | If the goods move to an alternative location, and a further arrival is sent, then this message needs to be validated. |
| EXT.RISK | Risk | The declaration data is passed to risk assessment which may trigger relevant control actions to be performed.<br><br>CSPs will be notified at this point of the control routing, if an EAA or EAL transaction has been submitted. |
| EXT.CTRL | Initiate and Perform Control | If the risk assessment identifies any potential actions to take place, this sub-process manages the tasks to be completed. A documentary control notification and/or a physical control notification may be sent to the submitter at this point.<br><br>Documentary controls will remain open for a maximum of 1024 days (3 years) before being referred to a caseworker for manual review. Physical/ALVS controls will remain open for a maximum of 180 days. |
| EXT.POS | Determine Customs Position | Following receipt of control results, CDS will take a customs position on the exit movement. The decision is made to either release the goods for exit or seize the goods. |
| EXT.CLR | Clear Declaration | The final process handles the clearance or release of goods for exit. The trader and CSP is notified of the customs decision at this point. |

### Conditions for a DUCR

Most conditions are covered with the regex but also additional checks in code for the additional conditions not covered by the regex.

| Condition | Covered by Regex |
| --- | --- |
| All alphas used must be upper case | Yes |
| First character to be a number | Yes |
| Second and Third character to be alpha (upper case only) | Yes |
| 16th digit to be a hyphen | No  |
| Post the hyphen any character is allowed but any alphas must be upper case | No  |
| There must be between 6 and 32 characters after the hyphen | Yes |

Any additional checks not covered by the regex will need to be validated by additional code or regexes.

## Exports Message Flow

The end-to-end sequence diagrams can be found in CDS End to End Sequence Diagrams \[2\] and are therefore not replicated here.

The relevant sequence diagrams that relate to Export Declaration Submission are:

- Submit Pre-Lodged Declaration
- Submit Arrived Declaration
- Submit EAA
- Submit Goods Arrival
- Submit EAC
- Submit CST
- Submit EDL
- Submit Amendment Request
- Submit Invalidation Request
- Submit Supplementary Declaration

Note that the listing order of these messages and requests is not significant.

## Key Functional Differences from CHIEF for Exports

This section outlines the key differences from CHIEF affecting Import declarations only, for key differences affecting both Imports and Exports, please refer to section

### Declaration Amendment Restrictions

Unlike Imports, amendments can be received on a declaration past the point of a customs position being taken. This includes after clearance or release is given. However, once the goods are departed, any amendment request will be rejected.

### CSE submitted as an arrived declaration

Customs Supervised Export (formally Local Clearance Procedure) will no longer have a dwell time prior to automatic arrival. The process now will be for a declarant to submit an arrived declaration. This will trigger a risk response as to whether the goods are under physical, documentary or no control. Once the goods arrive at the port for exit, the declaration will be re-arrived, and therefore re-risked.

### Indirect Export Processing

Indirect exports are goods exported from one Member State that exit the EU via another Member State. Northern Ireland can act as the customs office of export or the customs office of exit. The Automated Export System (AES) implemented into CDS builds upon the existing Export Control System (ECS). Additionally, AES includes uplifts to automate processes through electronic message exchange between external domain systems, national domain systems and common domain systems. Declarations for Indirect Export are processed through the CDS system for Northern Ireland Traders.

#### Message Flow: Northern Ireland as the Office of Export

The trader submits an indirect export declaration to CDS with an Office of Exit code which is not NI or GB in D.E. 5/12

- This declaration is processed by CDS and following clearance from the Office of Export, an Anticipated Export Record (AER) IE501 is sent by AES (within CDS) to the Office of Exit in the CCN.
- When goods have exited the EU, the Office of Exit will send the exit results in the form of an IE518 message to the Office of Export in the UK.
- CDS will receive the IE518, record the exit details and then notify the submitting trader via a DMSEOG.

The ability to amend the declaration will follow the usual amendment process. However, once the goods are released and are in transit to the Other Member State (OMS), no further amendment should be made. Instead, the individual presenting the goods at the Office of Exit will be allowed to indicate any discrepancies with what was declared at the Office of Export. These discrepancies will then be notified to Northern Ireland Customs by the OMS systems through the IE518 message (Exit Results).

##### Cancellation and Amendment Message Flow

- Cancellation and amendment requests will continue to be processed up to the release of the goods.

#### Message Flow: Northern Ireland as the Office of Exit

An IE501 has been received into CDS from the Office of Export (other Member State).

- The trader responsible for exiting the goods submits a pre-lodged indirect export declaration into CDS quoting the EU MRN provided in the IE501 in D.E. 2/1:
  - ‘18I’ must be declared in D.E. 1/11 for the declaration to be considered indirect.
  - The Office of Exit must contain a Northern Irish code.
  - Only one EU MRN must be provided per indirect export declaration.
- When the goods arrive, via an EAL message submitted into CDS, the risking and clearance process will take place with CDS notifying the trader of the outcome as per BAU.
- When departure of the goods has occurred (via an EDL), CDS will notify the trader as per BAU and an IE518 message will be sent by CDS to the Office of Export notifying them of the exit results.

An indirect export declaration can not be consolidated into a MUCR using any means, including EAC’s or Declaration submission.

###### Cancellation and Amendment Message Flow

- Cancellations will continue to be processed up to departure.
- Amendment requests to change the EU MRN provided in D.E. 2/1 after the declaration has been successfully arrived will be rejected.

### Excise Movements and ARC Numbers

If the document relates to a previous procedure, the ARC number would go in DE 2/1. If it relates to the current/requested procedure (or if it requires decrementing/validation) it should be declared in DE 2/3.

For example, if an eAD is created to move the goods from the port to enter an excise warehouse using PC 0700, the eAD would go in DE 2/3.

These numbers cannot be re-used by another declaration, unless the declaration is rejected or invalidated.

### MUCR Validation

Where a MUCR is declared in 2/1 on a declaration, this will be validated by CDS. Validation is passed if the MUCR is either ‘open’, or non-existent. If the validation is passed, this will cause the declared DUCR to be consolidated into the declared MUCR. Therefore, it is important to ensure that the MUCR is valid, else if the MUCR is shut this will cause CDS to reject the declaration.

### Transit Movements

Where ‘TRANS’ is declared in 2/2 across ALL goods items on a declaration, the exit message is not expected to be received. The export movement will be closed down at clearance and there is no need for the trader to notify Customs of exit.

Note, the future introduction of the Automated Exports System (AES) will impact transit movements for Northern Ireland traffic. This document will be updated when this is implemented.

### Alternative Proof of Exit

Solution to be further defined, but traders will be able to submit alternative proof of departure/exit where a departure/exit message has not been received. This will trigger the exit confirmation message (DMSEOG notification) from CDS, if the proof is accepted by Customs.

### Invalidation of Export Declarations after 150 Days

Within CDS, when an export declaration has been cleared/released, but has not exited for 150 days, that declaration will be invalidated. Reminders (in the form of a DMSGER notification) will be sent to the submitting trader after 90 days by the CDS system. If an exit message is not received from the relevant Office of Exit, the trader has the option to submit an Alternative Proof of Exit (refer to Alternative Proof of Exit).

### Processing Northern Ireland Export Declarations for GVMS integration

Where an export declaration features a Northern Ireland Goods Location Code (in D.E. 5/23) in which GVMS operates, an AI statement code of ‘RRS01’ is required to be provided in D.E.2/2 at the declaration header level. If this is not provided, CDS will reject the declaration.

**Note** – if an AI statement code of RRS01 is provided for a declaration with a Northern Ireland Goods Location Code in D.E. 5/23 that GVMS does not operate in, the declaration will be rejected. For the current NI GVMS Location Codes, please refer to the _CDS Codelists_ document. For further information on completing the declaration, refer to the latest _CDS Technical Completion Guide_.

### Processing GB Arrived Export Declaration at GVMS Category 1 Ports

Following the end of Staged Customs Controls in Jan 2022, most RoRo ports will be required to operate a full standard (pre-lodged) exports process, except for the GB ports that are identified as Ports with Reduced Capacity (Category 1). These locations will only allow Arrived Export declarations, so that goods selected for risk are checked and examined at an inland border facility, prior to physically arriving at the port.

For these goods to successfully pass, the declaration must include AI statement code ‘RRS01’ in D.E. 2/2 and authorisation type code ‘EXRR’ in D.E. 3/39, followed by the traders EORI number. The goods location code submitted in the declaration (D.E. 5/29) and departure message (EDL – Export Departure) submitted by GVMS **must** match. Alternatively, CSE authorisation may be used.

The current valid list of goods locations (submitted in D.E. 5/23 within the declaration form) that allow GB Arrived Exports are as follows:

- Dover = **GBAUDVRDOVDVRGVM**
- Eurotunnel = **GBAUEUTEUTEUTGVM**
- Dover/Eurotunnel = **GBAUDEUDEUDEUGVM**
- Holyhead = **GBAUHLYHLDHLYGVM**
- Liverpool = **GBAULIVLIVLIVGVM**
- Heysham = **GBAUHYMHEYHYMGVM**
- Fishguard = **GBAUFISFISFISGVM**
- Pembroke = **GBAUPEDMILPEDGVM**

#### DUCR presentation at GVMS Category 1 Ports

Owing to the Arrived Export declaration solution at GVMS Category 1 Ports, the GMR (Goods Movement Reference) will be checked commercially before goods are allowed to ship from the ports. However, whilst an EAL message is not required at Category 1 Ports, these movements will still receive an actual departure message via GVMS into CDS. If for any reason the DUCR in question is rejected during the declaration lifecycle, the re-use of the same DUCR is permissible across both CDS and GVMS.

#### DUCR presentation at GVMS Non-Category 1 Ports

Goods destined to export through a GVMS controlled Non-Category 1 Port will invariably be in a pre-lodged state and will require the submission of an EAL message. The exception to this is the re-arrival of CSE-based declarations. If the goods are rejected upon arrival at the GVMS controlled Non-Category 1 Port, current processing will require a new declaration and DUCR if goods are to be diverted to another port location.

#### Re-presentation of goods (previously ‘rejected upon arrival’ at GVMS Non-Category 1 Port) at GVMS Category 1 Ports

In rare edge cases, goods may get diverted from different GVMS controlled port locations. In all cases, where diversion occurs, the trader must create a new declaration and DUCR.

### Redirection to RoRo ports

Where an export declaration has been submitted initially for an export location of a non-RoRo port and the goods have subsequently been directed to a RoRo port, a declaration amendment should be made to;  

- Include the AI statement code in the declaration header D.E. 2/2 = ‘RRS01’
- Update the goods location code to reflect the revised (RoRo) location in D.E. 5/23.

# Additional Messages

Additional messages are messages that instruct CDS to perform a function related to an existing declaration: amendment, invalidation (cancellation), or goods arrival. They conform to the WCO Declaration schema, but only require a limited number of fields to be completed. They consist of:

- Standard header data (e.g. FunctionCode, FunctionalReferenceID, etc.)
- AdditionalInformation element(s)
  - This contains the free text reason for the amendment/cancellation
  - The free text being provided supplements a reason code that is provided in the Amendment element. The AdditionalInformation element will therefore need to contain pointers that point at the Amendment element that contains the corresponding reason code.
  - For a cancellation, there will only be one Amendment element to point to. For amendments, there will need to be one free text reason per Amendment element, though they may all contain the same value. Where multiple Amendments are being made, each free text reason is to be provided in its own AdditionalInformation element that points to its corresponding Amendment element.
- Amendment element(s)
  - This contains the reason code for the amendment/cancellation. CDS supports either a single reason code being applied across all amendments or a specific reason code being applied to each amendment. The requirements as to which approach is used will be defined by the software being used to construct the amendment.
  - If amending, the Amendment element also has to point at the element within the declaration that is being amended
- Declaration data being amended (does not apply to cancellations)
  - If data is being added or updated, the new value(s) will need to be included in the message to form part of the amendment instruction
  - Sequence numbers must be maintained – an example of this can be seen in section .

Details on how to complete an additional message can be found in the CDS Additional Message Technical Completion Matrix \[7\].

When submitting additional messages, a new conversation ID will be provided in the synchronous response that will allow notifications related to the additional message to be differentiated from those relating to the declaration. For example, if a pre-lodged declaration is cancelled one of two scenarios may occur:

1. The cancellation is successful, and the trader will receive a DMSRCV and a DMSREQ with the conversation ID of the additional message, followed by a DMSREJ with the conversation ID of the declaration
2. The cancellation is unsuccessful, and the trader will receive a DMSREJ with the conversation ID of the additional message.

## Amendments

With requests for amendment, CDS can handle the addition and deletion of complex objects but there are certain considerations to be taken into account (The "lowest" level elements, that do not have any child elements "below" them, are "leaf" nodes that are pointed to using the tag identifier. All higher level or ‘parent’ elements - i.e. "above" the leaf nodes - are pointed to using the DocumentSectionCode, and are complex elements):

- Generally,when **adding** entire complex objects, i.e. a data element (e.g. Declaration/GoodsShipment/GovernmentAgencyGoodsItem/AdditionalDocument) that has child elements, a single amendment data element should be used that points to the parent rather than having multiple amendment data elements pointing to each child (Noting the limited number of complex data elements that cannot be directly pointed to in amendment requests, where all sub-elements/leaves must be pointed to – see data elements below the modifying or deleting data element bullet point).
- Certain XML elements are concatenated into a single value for validation. For example, in the complex element Additional Document, the Category Code and Type Code are concatenated. If **amending** both these elements that are concatenated, only one amendment data element is needed, pointing to either the Additional Document/TypeCode or AdditionalDocument/CategoryCode
- When **modifying** or deleting multiple data elements you can do so by adding the parent element however, there are certaiin data elements that do not allow you to do this. The modification or deletion of data elements currently needs to point to each child element separately because some parent elements cannot be modified or deleted directly.

If you point to the parent in an amendment (COR or GPN) additional message, then the entire parent will be replaced (however, note exceptions of parent elements that cannot be deleted at present, as listed below)

So, for example, if the "current" values (e.g. as given in the original declaration payload) has 3 child elements, and the amendment only has 2 of these child elements, the third child element will be deleted.

Hence if you are only amending specific child elements, then you should continue to use individual pointers for each child. If you are replacing the entire parent, point to the parent.

Parent elements that cannot at present be deleted include:

- - Declaration\\FunctionCode 017 (This cannot be amended anyway.)
    - Declaration\\Authentication 61B
    - Declaration\\Authentication\\Authenticator 12A
    - Declaration\\GoodsShipment\\Consignment 28A
    - Declaration\\GoodsShipment\\CustomsValuation 41A
    - Declaration\\GoodsShipment\\GovernmentAgencyGoodsItem\\AdditionalDocument\\Submitter 17B
    - Declaration\\GoodsShipment\\GovernmentAgencyGoodsItem\\AdditionalDocument\\WriteOff 22C
    - Declaration\\GoodsShipment\\GovernmentAgencyGoodsItem\\Commodity\\DutyTaxFee\\Payment 94A
    - Declaration\\GoodsShipment\\GovernmentAgencyGoodsItem\\Commodity\\InvoiceLine 79A
    - Declaration\\GoodsShipment\\GovernmentAgencyGoodsItem\\CustomsValuation 41A
    - Declaration\\GoodsShipment\\GovernmentAgencyGoodsItem\\GovernmentProcedure 70A
    - Declaration\\GoodsShipment\\GovernmentAgencyGoodsItem\\ValuationAdjustment 39B
    - Declaration\\ObligationGuarantee\\GuaranteeOffice 14C

This applies both when using an amendment message, as well as when including amendments within a Goods Presentation Notice (GPN).An update, to enable all parent elements to be modified or deleted, is being considered.

- When pointing to an element, that pointer covers both the element value and its attributes. This means that if only one of those is being amended, the other will still need to be provided. For example, where D.E. 8/6 (Statistical Value) is being amended, even if the currency ID has not changed, it will need to be included in the &lt;StatisticalValueAmount&gt; tag within the additional message to ensure that it is persisted in the updated version of the declaration.

#### Imports-Specific Considerations

- Amendment requests can only be granted before the end of the 10-minute dwell time
- The dwell time does not restart if an amendment request is submitted and granted.
- Where an amendment has been rejected during the dwell time: the declaration will not progress past the dwell time onto tax calculation and clearance until an amendment additional message is submitted that is accepted and processed by CDS.
  - This is to ensure that where a trader has indicated that they would like to amend their declaration, but they happen to have made a mistake and the additional message is rejected, declaration processing does not continue on a version of the declaration that is potentially incorrect.
  - If an amendment is required, but the amendment additional message was rejected, a valid additional message would allow the declaration process to continue towards clearance.
  - If an amendment isn’t actually required, but the amendment additional message was mistakenly submitted and rejected, a ‘nil’ amendment would allow the declaration process to continue towards clearance.

#### Exports-Specific Considerations

- Export declarations can be amended up until the goods have been departed.
- Export declarations cannot be amended after the Exit Message has been received.
- If an amendment to an Export declaration has been rejected (by means of receiving a DMSREJ), if the requested change is subsequently not needed to be applied, there is no requirement to submit a nil amendment message.

### Data Element amendment restrictions

Some data elements are unable to be amended after a declaration has been submitted.

Data elements unable to be altered (or removed) within an amendment across all declaration types are;

• Declaration Type (DE 1/1, DE 1/2) – i.e. an Imports declaration cannot be amended to be an Exports declaration, likewise a supplementary declaration cannot be amended to be an arrived frontier declaration.

• Functional Reference ID (DE 2/5)

• MUCR and DUCR values\*

• Removal of any field or sub-field required by a customs procedure specified in the declaration (i.e. a mandatory data element as defined in the Technical Completion Matrix \[5\]).

Amendments to any of the above data elements will result in the amendment request being rejected by CDS.

In addition to the data elements stated above, where declarations are under blocking control, amendments may require approval from a HMRC Caseworker - and may result in the request being rejected for other, operational, reasons.

\*Note: MUCR and DUCR values may be amended on an import declaration, however we strongly advise against this unless it is to correct an inventory mismatch confirmed in a DMSRCV message. For export declaration amendments it is crucial a MUCR is not added via an amendment, as these changes will not make their way to ILE. Moreover, if this is done there will be no rejection message will be sent when you submit a MUCR/DUCR via a declaration amendment (This is not applicable to import declarations). Please refer to the ILE service design for the applicable processes and requirements.

### ‘Nil’ Amendments

Nil amendments are used in three scenarios:

- Confirmation that the FEC check will not result in an amendment to the declaration and that the originally declared values are correct. Note that as FEC checks are now non-blocking, this will not be needed to progress the declaration, but does provide supporting information to demonstrate compliance.
- Request for CDS to re-attempt inventory linking with a CSP as the inventory system been updated.
- Confirmation that the current version of the declaration is correct following a rejected amendment request during the dwell time.
- Confirmation that the current version of the declaration is correct following any rejected amendment request for a declaration under customs control.

The nil amendment is effectively an amendment submission that is not intended to change the data within a declaration. The expected format is shown in an example in section . Note that our rules treat actual amendments and nil amendments the same for validation purposes.  

### Amending a declaration to add Goods Items

Additional Goods Items may be added to a declaration, depending on its stage within processing.

For Imports declarations;

- Pre-lodged declarations can be amended to include additional items prior to arrival.
- For arrived declarations this is possible for declarations not under control.
- Where an arrived declaration is under control, a Customs decision will be made to determine if this request is permissible or not.

For Exports declarations;

- Pre-lodged declarations can be amended to include additional items prior to departure.
- For arrived declarations this is possible for declarations not under control.
- Where an arrived declaration is under control, a Customs decision will be made to determine if this request is permissible or not.
- It is possible to amend an export declaration up until the goods have departed.

For both Imports and Exports, when submitting an amendment message requesting to add goods items, the Goods Item Quantity and Invoice Amount values should be amended accordingly for the amendment to pass validation. To avoid schema errors when submitting a COR (amendment additional message), the &lt;GoodsItemQuantity&gt; amended value should be provided **before** the &lt;AdditionalInformation&gt; elements.

The goods item number(s) for the new item(s) should continue in sequence following on from those in the originally submitted declaration. Where item(s) have been removed from the declaration those goods item numbers may not be reused for new items and the next available in sequence should be used (refer to for examples).

### Amending a declaration to remove Goods Items

It is possible to submit an amendment to request the removal of one or more goods item(s) from a submitted imports declaration.

For the removal of Goods Item(s) message to be accepted by CDS;

- Valid pointer(s) to the goods item(s) requested to be removed by the amendment
- The _Total Number of Items_ data element must also be updated by the trader within the amendment, to match the number of items remaining on the amended declaration. To avoid schema errors when submitting a COR (amendment additional message), the &lt;GoodsItemQuantity&gt; amended value should be provided **before** the &lt;AdditionalInformation&gt; elements
- At least one Goods Item must remain on the amended declaration.

Where an amendment to remove goods items accepted by CDS, related quota or license reservations will be updated accordingly.  
<br/>**Notes**

- The Goods Item Number relating to the removed item(s) cannot be reused.  
    <br/>e.g. If a declarant has submitted a declaration containing two goods items and has successfully amended a declaration to remove goods item two, if the declarant was to make a further amendment to add a new goods item to that declaration, the goods item sequence number should be ‘3’.
- The sequence numbers of other items in the declaration will not change as a result of an item being removed.  
    <br/>e.g. If a declarant has submitted a declaration containing two goods items and has successfully amended a declaration to remove goods item one, the sequence number of the remaining goods item will remain as ‘2’ and any future amendments needed to be made to that remaining item should include a pointer to goods item two.

Declarations subject to a blocking control may require approval from a HMRC Caseworker before requests to remove goods items are accepted.

### Changes to Additional Information Header Field

The additional message contains two fields “Additional Information” and “Amendment”. The additional information field, section 1 in the XML sample below, does not directly relate to any values inputted in DE 2/2 on the trader’s declaration. This field specifies to CDS the Statement type code AES (Amend Goods Description) and the value entered in the subfield “Statement Description” provides context as to what values were changed by this amendment.

Introducing Additional Information into DE 2/2 at header level (AI statement RRS01 for GVMS) does not conflict with the existing use of Additional Information Statement (to specify the reason for an amendment) as part of amendment functionality. An amendment using AI code AES with an AI statement description e.g. ‘Amend Goods Description’ does not have any impact on the content of the declaration, so will not overwrite the AI statement RRS01. CDS will only ever amend the objects specified by the pointer in the amendment object of the additional message.

Note that when submitting an amendment to Additional Information (AI) objects at the header level, the order of AI objects provided in the amendment message must match the order provided in the original submitted declaration.

The amendment field, section 2 in the XML sample below, is the only field that changes the values on the trader’s declaration and only pointers to DE 2/2 within this field are able to change the additional information data specified on the trader’s declaration.

**1**.

**2**.

- The AES AdditionalInformation elements (in the first section of the COR) do not update the declaration, they only inform CDS of the reason for the amendment. Each AES element should point to the corresponding Amendment element of the COR that it describes.
- the Amendment elements (in the second section of the COR) point to the data element to be amended (i.e. to the relevant section within the existing declaration)
- the Amended Data elements (in the third section of the COR) provide the new (amended) values for the amended data elements.

So, for example,

- if you are amending 3 data elements of a declaration, there would be 3 Amendment elements in the COR, each pointing to one of the declaration DEs to be amended
- If you are only amending the fourth data element of a particular type in the existing declaration, then in the second section of the COR, the Amendment element SequenceNumeric would have a value of 4 to indicate this.
- as regards the sequence number referencing within the Amended Data section of the COR
    1. if the declaration data element being amended also includes a SequenceNumeric, then the Amended Data in the COR too should have the appropriate value (e.g. 4 in the above example) (see DSSD section "6.3.5.1 Complex Amendment Example 1")
    2. if the declaration data element being amended has an implicit sequence number, rather than an explicit one as for the goods item, then empty placeholder data elements need to be included in Amended Data (third section of the COR) so that the system can maintain this implied sequence for the existing declaration objects. (see DSSD section "6.3.5.2 Complex Amendment Example 2")

To change a Document number 3, inside Commodity 2" - "Name of Document number 3 ('CDS Waiver' in this case) to something else. See below.

&lt;Amendment&gt;

&lt;ChangeReasonCode&gt;28&lt;/ChangeReasonCode&gt;

&lt;Pointer&gt;

&lt;SequenceNumeric&gt;1&lt;/SequenceNumeric&gt;

&lt;DocumentSectionCode&gt;42A&lt;/DocumentSectionCode&gt;

&lt;/Pointer&gt;

&lt;Pointer&gt;

&lt;SequenceNumeric&gt;1&lt;/SequenceNumeric&gt;

&lt;DocumentSectionCode&gt;67A&lt;/DocumentSectionCode&gt;

&lt;/Pointer&gt;

&lt;Pointer&gt;

&lt;SequenceNumeric&gt;2&lt;/SequenceNumeric&gt;

&lt;DocumentSectionCode&gt;68A&lt;/DocumentSectionCode&gt;

&lt;/Pointer&gt;

&lt;Pointer&gt;

&lt;SequenceNumeric&gt;3&lt;/SequenceNumeric&gt;

&lt;DocumentSectionCode&gt;02A&lt;/DocumentSectionCode&gt;

&lt;TagID&gt;D028&lt;/TagID&gt;

&lt;/Pointer&gt;

&lt;/Amendment&gt;

&lt;GoodsShipment&gt;

&lt;GovernmentAgencyGoodsItem&gt;

&lt;SequenceNumeric&gt;2&lt;/SequenceNumeric&gt;

&lt;AdditionalDocument/&gt;

&lt;AdditionalDocument/&gt;

&lt;AdditionalDocument&gt;

&lt;Name&gt;something else&lt;/Name&gt;

&lt;/AdditionalDocument&gt;

&lt;/GovernmentAgencyGoodsItem&gt;

&lt;/GoodsShipment&gt;

### Amending a declaration to change Additional Procedure Codes (APCs)

The way pointers are used with the Customs Procedure Code (CPC) components, namely DE 1/10 (RPC and PPC) and DE 1/11 (APC), is described below.

There must always be exactly one GovernmentProcedure that has both CurrentCode and PreviousCode child elements. This GovernmentProcedure maps to the RPC and PPC as follows

&lt;GovernmentProcedure&gt;

&lt;CurrentCode&gt;40&lt;/CurrentCode&gt; &lt;!-- Requested Procedure Code - RPC --&gt;

&lt;PreviousCode&gt;00&lt;/PreviousCode&gt; &lt;!-- Previous Procedure Code - PPC --&gt;

&lt;/GovernmentProcedure&gt;

There can be one or more GovernmentProcedure data elements that have only the CurrentCode child element. These map to the APCs.

&lt;GovernmentProcedure&gt;

&lt;CurrentCode&gt;000&lt;/CurrentCode&gt; &lt;!-- Additional Procedure Code - APC --&gt;

&lt;/GovernmentProcedure&gt;

Note that both RPC and APCs have the same WCO path; they are distinguished by the sequenceNumeric of the corresponding GovernmentProcedure.

- For RPC, sequenceNumeric for 70A (GovernmentProcedure) is 1.
- For APCs, sequenceNumeric for 70A (GovernmentProcedure) starts at 2.

However, in the DMSREJ when there is an error with the APCs, the sequenceNumeric for 70A is returned starting with 1.

This is explained in detail in section 6.2.2.3.2 Use of Pointers for DE 1/10 (RPC, PPC) and DE 1/11 (APCs)

### Use of Pointers for DE 1/10 (RPC, PPC) and DE 1/11 (APCs)

The pointer usage required when amending an APC is slightly different to the pointer usage when errors are reported by CDS via the DMSREJ.

**When amending an APC through an additional message (COR or GPN)**

If the trader enters the below on their declaration:

&lt;GovernmentProcedure&gt;

&lt;CurrentCode&gt;40&lt;/CurrentCode&gt; &lt;!-- Requested Procedure Code - RPC --&gt;

&lt;PreviousCode&gt;00&lt;/PreviousCode&gt; &lt;!-- Previous Procedure Code - PPC --&gt;

&lt;/GovernmentProcedure&gt;

&lt;GovernmentProcedure&gt;

&lt;CurrentCode&gt;000&lt;/CurrentCode&gt; &lt;!-- Additional Procedure Code - APC --&gt;

&lt;/GovernmentProcedure&gt;

Because we have two objects of GovernmentProcedure, we use the sequenceNumeric in WCO 70A to point to the correct object we are amending

| **Amending Field / DE** | **Pointers** |
| --- | --- |
| RPC | 42A/67A/68A\[1\]/70A/166 or 42A/67A/68A\[1\]/70A\[1\]/166 |
| PPC | 42A/67A/68A\[1\]/70A/161 |
| APC | 42A/67A/68A\[1\]/70A\[**2**\]/166 |
| RPC and APC | 42A/67A/68A\[1\]/70A/166 or 42A/67A/68A\[1\]/70A\[1\]/166 (for the RPC) and<br><br>42A/67A/68A\[1\]/70A\[**2**\]/166 (for the APC) |

If no sequence number is used for 70A then CDS defaults to sequence number 1 and likewise sequence number 1 refers to as the first governmentProcedure.

**When reporting an error with an APC, through the DMSREJ**

However if the 1/10 and 1/11 combination is incorrect and we reject on the business rules, then the DMSREJ is sent with the below pointers:

| **Erroneous Field / DE** | **Pointers** |
| --- | --- |
| RPC | 42A/67A/68A\[1\]/70A/166 |
| PPC | 42A/67A/68A\[1\]/70A/161 |
| APC | 42A/67A/68A\[1\]/70A\[**1**\]/166 |
| RPC and APC | 42A/67A/68A\[1\]/70A/166 (for the RPC) and<br><br>42A/67A/68A\[1\]/70A\[**1**\]/166 (for the APC) |

Here APC code is pointed at using sequence number 1, but if the trader wanted to amend this same field then they would have to use sequence number 2.

If the trader uses sequence number 1 for GovernmentProcedure when amending, then CDS thinks it is looking at the RPC not the APC and will delete the RPC.

**Specific notes on amendments to DEs with multiple instances in that DE (for example, DE 2/3, DE 2/2, etc):**  
•    CDS will allow an instance of an element to be removed from a declaration (for example, an additional document in DE 2/3), as long as it is not combined with adding or amending another instance in the same DE due to an empty placeholder conflict.  
•    Removal of an instance required for the declaration to pass validation (for example, removal of a document code required to satisfy a measure applied to the Commodity code on the same goods item), will result in a rejection of the amendment.  
•    If there is an intention to replace an existing instance in such a DE with a different instance in the same DE (for example, removing one document code and adding another document code in DE 2/3), it is recommended that the amendment overwrites & replaces the data for the original instance with the data for the new instance (for example, amend the data in the original sequence/placeholder in DE 2/3 with the new data).

## Cancellation specific considerations

The following needs to be considered for cancelling declarations:

For Imports declarations;

- Declarations can be cancelled up until clearance has been granted.
- A DMSINV trader notification will only be received if the declaration has already been legally accepted (i.e. a DMSACC has been received). If this is not the case (e.g. for a pre-lodged declaration), a DMSREJ will instead be sent to notify of the cancellation.
  - Where a pre-lodged declaration is successfully cancelled, this can be identified by the DMSREJ being received with the same conversation ID as that of the declaration, and without containing any error codes.

For Exports declarations;

- Declarations can be cancelled up until departure and cannot be cancelled if the exit message has been sent for exports.
- A DMSINV trader notification will only be received if the declaration has already been legally accepted (i.e. a DMSACC has been received). If this is not the case (e.g. for a pre-lodged declaration), a DMSREJ will instead be sent to notify of the cancellation.
- Where a pre-lodged declaration is successfully cancelled, this can be identified by the DMSREJ being received with the same conversation ID as that of the declaration, and without containing any error codes.

A declaration cannot be invalidated while in an amendment state following an unsuccessful/rejected amendment, however if the trader does not want to progress with that declaration and intends to cancel the declaration, rectifying the unsuccessful/rejected amendment will probably mean the declaration progresses to a customs position (to clearance if payment/security is successfully taken, or if payment/security not required).

The only options if the trader intends to cancel/invalidate the declaration is:  

1. If the declaration requires payment/security to be taken then make an amendment that fixes the rejection, but remove all MoP details so the declaration goes to immediate payment at the end of the dwell time - the declaration can then be invalidated (this is not a fool-proof option though as some decs do not need payment/security to clear);
2. Abandon the declaration and resubmit (where required) with the correct details entered on the declaration.

However, option 2 will require amending the inventory if the dec used a UCR/UCN that was successfully linked to the stuck declaration - this cannot be resolved via CDS though and must be undertaken via the CSP concerned.

## Non-Inventory Goods Arrival specific considerations

For declarations that are inventory-linked, they will be arrived by the CSP inventory system through an arrival notification submitted into the relevant inventory linking interface.

For non-inventory-linked declarations, there is a separate arrival message (in a different format) that can be submitted where there is no inventory system. Details of the Non-Inventory-Linked Port (NILP) arrival process is described in the End to End Sequence Diagrams document \[2\]. Please note that trying to arrive an inventory-linked declaration with the non-inventory message will cause a mismatch with the inventory system and will potentially lead to the declaration being rejected.

For the non-inventory goods arrival, the following need to be considered:

- This function is only to be used for non-inventory linked declarations. Using this with any other type of declaration will cause a delay to clearance.
- Goods arrival messages are required to inform CDS that goods associated with a pre-lodged declaration have arrived and are available for inspection. This will lead to e.g. a “type D” declaration to change to a “type A” declaration.

In certain scenarios, where required, amendments to the pre-lodged declarations can be provided within the goods arrival message, rather than being included as part of a normal amendment additional message. An example of this is outlined in section .

- If any amendments are made to declaration data (regardless if inventory-linked or not, and if the provided data has changed from the initial pre-lodged declaration) via a goods arrival message, the trader will receive a DMSRES trader notification.
  - If the goods are now at a location that was not specified on the pre-lodged declaration, D.E. 5/23 Goods Location can be amended within the goods arrival message.
    - As 5/23 data components are distributed across multiple elements, separate Amendments are required for each updated element.
  - If the nationality of the means of transport crossing the border has changed since the declaration was pre-lodged, D.E. 7/15 Nationality of active means of transport crossing the border can be amended within the goods arrival message
  - The trader should reflect the changes advised in the DMSRES in their own records, so their declaration data aligns with that held by HMRC.  
        **Note**, for Inventory Linked declarations, the goods arrival message sent to CDS by the CSP does not include a Change Reason, therefore a change reason will not be included in the resulting DMSRES notification message.
- The error conditions for the Goods Arrival are specified in the Inventory Linking Import Service Design \[3\] and Inventory Linking Exports Service Design \[8\].

## Non-CSP specific considerations

Non-CSPs must not include a MCR (inventory reference) when submitting an import declaration to CDS, otherwise the declaration will not progress.

The presence of the MCR will cause CDS to believe the submitter is a CSP. CDS will then attempt to exchange inventory linking messages (validateMovementRequest) with the submitting software.

Since the non-CSP software is not setup to process this notification - nor send the corresponding validateMovementResponse message back to CDS - it will appear to the user the declaration has failed processing, i.e. no DMSxxx message will have been sent.

Meanwhile, CDS will remain in a waiting state for the movement response and will therefore not progress the declaration beyond this step. This is expected behaviour of CDS.

# Trader Notifications (Imports / Exports)

## Types and Definitions

Below are the possible trader notifications that CDS will send. The End to End Sequence Diagrams \[2\] feature these in context of the relevant scenarios where CDS may send them. Where a notification type is applicable to only Imports or Exports, this is indicated.

Internally, we attempt to issue notifications in the order they are generated.

Assuming you are using the push interface, this sequencing can, however, be impacted by two factors outside our control:

1. If we have trouble sending a specific notification to you, we will back off for a few seconds before trying again. Any other notifications to you, however, won't be paused and may go straight through
2. The internet may send different notifications via different routes to you. Larger notifications are also likely to be split into smaller parts and again, may be sent via different routes. This may result in notifications being delivered in a different order.

We issue notifications as soon as they are generated. Multiple notifications may be issued within seconds.

If you use the pull interface, you can pull notifications at whatever timing and in whichever order you choose.

| **DMS Code** | **Notification Type** | **Description** | **Sub-process Found** | **Mapping to CHIEF Equivalent** |
| --- | --- | --- | --- | --- |
| 01  | DMSACC | With this message the submitter is informed that the declaration has been legally accepted | Validate Declaration | Contained within E2 and X2 |
| 02  | DMSRCV | With this message the submitter is informed that the message has been registered. This can apply to pre-lodged declarations as well as additional messages. | Validate Declaration | H2/P2 |
| 03  | DMSREJ | With this message the submitter is informed that the received message has been rejected. This can apply to declarations and additional messages. | Validate Declaration | Cancellation Refusal: N3/S3 |
| 05  | DMSCTL | This message informs the submitter that Customs intends to physically examine the goods. This will only be received when the goods are on hand. | Initiate Control | E1 (Imports) / X1 (Exports) |
| 06  | DMSDOC | This message informs the submitter that the declarant needs to present one or more documents related to the declaration. | Perform Control | Contained within E2 |
| 07  | DMSRES | The submitter is informed that corrections have been applied to the declaration by the trader, or as a result of physical inspection by customs (latter not likely to be triggered). | Clear Declaration | N/A |
| 08  | DMSROG | This message informs the submitter that the goods can now be released. Note that this is different to clearance as it implies that the debt calculated is not yet finalised. | Clear Declaration | Currently implied by N5 / IVL notification |
| 09  | DMSCLE | This message informs the submitter that the declaration is now cleared, and by implication, the goods can be released. | Clear Declaration | Contained in N5 report if non-inventory linked.<br><br>If inventory linked, trader will receive a notification from the inventory system. |
| 10  | DMSINV | This message informs the submitter that the declaration has now been cancelled. This can be as a result of a trader-initiated invalidation, or system-initiated. | Invalidate Declaration | Cancellation Approval: N4/S4<br><br>Goods Disposal Advice: S9 |
| 11  | DMSREQ | With this message the submitter is informed of the acceptance or denial of the additional message submitted to customs. In the case of a denial, the reason is indicated. | Handle Additional Message | N3/S3, N4/S4 |
| 13  | DMSTAX | **IMPORTS ONLY.** The submitter is informed of the indicative calculation or duties liable with this message.<br><br>Indicative calculations, sent before the dwell time, provide traders with opportunity to preview the duty calculation and make amendments to the declaration as necessary to rectify completion errors, or cancel and resubmit.<br><br>Calculations made following the debt calculation stage (after the dwell time concludes) will confirm the payable amount and either inform on what has been reserved against a deferment account or similar method of payment. Alternatively, it can be used as a payment instruction for immediate methods of payment. In the immediate payment scenario, a payment reference is also provided. | Handle Customs Debt | Contained within E2 |
| 14  | DMSCPI | **IMPORTS ONLY.** This message informs the submitter of the insufficient balance against a deferred method of payment. The relevant financial party will need to take action to rectify the situation before the goods can be released. | Handle Customs Debt | E9 and X9 |
| 15  | DMSCPR | **IMPORTS ONLY.** This message is a reminder regarding the need to take action on the insufficient balance against a deferred method of payment. It can also be used as a reminder to make an immediate payment. | Handle Customs Debt | N/A |
| 16  | DMSEOG | **EXPORTS ONLY.** The message informs the submitter that the goods have now exited the customs union. | Monitor After Release | Indicated by ECS via IE518 |
| 17  | DMSEXT | This message informs the submitter that the declaration needs to be handled manually. Only used in exceptional circumstances where the system cannot handle the declaration. | Handle Irregularities | N/A |
| 18  | DMSGER | **EXPORTS ONLY.** This message informs the submitter that the exit results have not yet been received by the Export system. This could be a trigger to submit an alternative proof of exit. | Monitor After Release | S0  |
| 50  | DMSALV | **IMPORTS ONLY.** This message informs the submitter that a decision has been made by Defra that will delay clearance of the goods. | Perform Control | Contained within E0 |
| 51  | DMSQRY | This message informs the submitter that a query has been raised on the declaration by Customs, advising that caseworker has sent a secure message and is awaiting action by the submitter.<br><br>The DMSQRY notification message does not include the source or contents of the secure message - this is only accessible via the user interface of the Secure File Upload service. | Perform Control | N/A |

## Notification Message Structure

### Country specific message envelope

The schema for this envelope is called:

xsd/DocumentMetadata_2_DMS.xsd

The root element for this envelope is:

**Metadata**

and use the namespace:

**urn:wco:datamodel:WCO:DocumentMetaData-DMS:2**

| **Element** | **Cardinality** | **Value** |
| --- | --- | --- |
| **Metadata** |     |     |
| WCODataModelVersionCode | 1..1 | “**3.6**” |
| WCOTypeName | 1..1 | “**RES**” |
| ResponsibleCountryCode | 0..1 | “**”** |
| ResponsibleAgencyName | 0..1 | “”  |
| AgencyAssignedCustomizationCode | 0..1 | Not used |
| AgencyAssignedCustomizationVersionCode | 0..1 | “”  |

### Notification Contents

CDS notifications will always include timestamps in UTC throughout the year (which will be one hour behind during BST). The timezone is indicated by 'Z' being added to the payload. This has been an implementation we agreed with Software Houses.

If this needs to be converted into local time (BST), the timestamp will need to be converted into local time by Software Developers based on the timezone indicated.

Note that the acceptance date is indicated by setting the time component to zeros. For example,

an acceptance date 03-04-2023 00:00:00, which in BST is 20230403000000(Z+1), is shown in UTC as 20230402230000Z.

So at first glance, the acceptance date would give the impression of being one day early.

#### DMSACC

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSACC=01 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-Error** | 0..\* | Used to return validation results (validation errors, but potentially also other validation results such as FEC warnings). |
| \--Error/ValidationCode | an..8 |     |
| \--Error/Description |     |     |
| **\--Pointer** | 0..\* |     |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| Pointer/TagID | an..4 |     |
| **\-AdditionalInformation** | 0..\* | Might be used to report additional details regarding to validation results. |
| \--AdditionalInformation/StatementCode | an..17 | Additional validation information, coded. |
| \--AdditionalInformation/StatementDescription | an..512 |     |
| \--AdditionalInformation/StatementTypeCode | an..3 | A value from code list ValidationInformationType |
| **\--Pointer** |     |     |
| \---DocumentSectionCode |     |     |
| \---SequenceNumeric |     |     |
| \---TagID |     |     |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/AcceptanceDateTime | an..35 |     |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration (the MRN). |
| \--Declaration/VersionID | an..9 |     |

#### DMSRCV

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSRCV=02 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-Error** | 0..\* | Used to return validation results (validation errors, but potentially also other validation results; such as FEC warnings). |
| \--Error/ValidationCode | an..8 |     |
| \--Error/Description |     |     |
| **\--Pointer** | 0..\* |     |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| \---Pointer/TagID | an..4 |     |
| **\-AdditionalInformation** | 0..\* | Might be used to report additional details regarding to validation results. |
| \--AdditionalInformation/StatementCode | an..17 | Additional validation information, coded. |
| \--AdditionalInformation/StatementDescription | an..512 |
| \--AdditionalInformation/StatementTypeCode | an..3 | A value from code list ValidationInformationType. |
| **\--Pointer** | 0..\* |     |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| \---Pointer/TagID | an..4 |     |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration or additional message. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration (the MRN). |
| \--Declaration/VersionID | an..9 |     |

#### DMSREJ

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSREJ=03 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-Error** | 0..\* | Used to return validation results (validation errors, but potentially also other validation results). |
| \--Error/ValidationCode | an..8 |     |
| \--Error/Description |     |     |
| **\--Pointer** | 0..\* |     |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| \---Pointer/TagID | an..4 |     |
| **\-AdditionalInformation** | 0..\* | Might be used to report additional details regarding to validation results. |
| \--AdditionalInformation/StatementCode | an..17 | Additional validation information, coded. |
| \--AdditionalInformation/StatementDescription | an..512 |
| \--AdditionalInformation/StatementTypeCode | an..3 | A value from code list ValidationInformationType. |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration or additional message. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration (the MRN). Will contain “FAKEID” if an MRN has not been generated. |
| \--Declaration/RejectionDateTime | an..35 |     |
| \--Declaration/VersionID | an..9 |     |

Note that in certain cases where DMS has not been able to generate an MRN, for example if a declaration has been mistakenly submitted with an amendment FunctionCode, a dummy value of “FAKEID” will be used in the DMSREJ notification.

##### AdditionalInformation element DMSREJ clarification

Error blocks can have further information associated to them contained in the AdditionalInformation object.  
AdditionalInformation objects can be used to supplement error codes with further detail on what has failed.

These are optional and are dependent on the error code. They are used for Tariff and Inventory Linking validation results. In addition to the goods item pointer (if there is a pointer raised within the Error), there can also be an AdditionalInformation element provided that includes more detail.

The Error indicates that something is wrong in the declaration, and the AdditionalInformation indicates what is wrong with the declaration. Each Error can have multiple AdditionalInformation elements relating to it, and as they are both defined as siblings in the schema, the WCO Pointer mechanism is used to link one to another.

While the pointers contained in the Error element point back to declaration elements, the pointers contained in the AdditionalInformation element point to the Error element within the trader notification that it relates to.

The CDS CodeList uses the WCO pointer mechanism to describe the specific data element to which the information relates to. This is used to identify where the issues occurred in the declaration when it is rejected. As such, to identify the specific problem to resolve, the submitting system will need to interpret the DMSREJ notification to understand the validationResult code, in addition to the data element that the code relates to.

Within CDS, the pointer mechanism is used in 2 ways:

• In Trader Notifications to identify reasons for rejection, or reasons for control;

• In AdditionalInformation object to indicate which element it is related to in the declaration.

Pointers included in a DMSREJ notification, are referring to elements found on a declaration.

XML Sample

&lt;\_2_1:AdditionalInformation&gt;

&lt;\_2_1:StatementDescription&gt;The customs value cannot be determined as the ItemChargeAmount, StatisticalValueAmount, AdValoremTaxBaseAmount and CustomsValueAmount are absent. Please provide at least one to determine the customs value.&lt;/\_2_1:StatementDescription&gt;  
&lt;\_2_1:StatementTypeCode&gt;1&lt;/\_2_1:StatementTypeCode&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:SequenceNumeric&gt;1&lt;/\_2_1:SequenceNumeric&gt;  
&lt;\_2_1:DocumentSectionCode&gt;07B&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:SequenceNumeric&gt;1&lt;/\_2_1:SequenceNumeric&gt;  
&lt;\_2_1:DocumentSectionCode&gt;53A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;/\_2_1:AdditionalInformation&gt;  
&lt;\_2_1:Error&gt;  
&lt;\_2_1:Description&gt;The customs value cannot be determined as the ItemChargeAmount, StatisticalValueAmount, AdValoremTaxBaseAmount and CustomsValueAmount are absent. Please provide at least one to determine the customs value.&lt;/\_2_1:Description&gt;  
&lt;\_2_1:ValidationCode&gt;CDS40011&lt;/\_2_1:ValidationCode&gt;

&lt;/\_2_1:Error&gt;

###### Pointers to WCO IDs CodesList 'WCO References (Declaration) Tab'

| **Element name** | **WCO ID** |
| --- | --- |
| **AdditionalInformation** | 03A |
| **Pointer** | 97A |
| DocumentSectionCode | 375 |
| SequenceNumeric | 006 |
| TagID | 381 |
| StatementCode | 226 |
| StatementDescription | 225 |
| (@languageID) |
| StatementTypeCode | 369 |

When pointers included in a DMSREJ notification, for example, are referring to elements found on a declaration, these pointers are WCO IDs found in the 'WCO References (Declaration) Tab'.

Pointer (97A) -> DocumentSectionCode (375)

Pointer (97A) -> SequenceNumeric (006)

Pointer (97A) -> Tag ID (381)

###### Pointers to WCO IDs CodesList 'WCO References (Trader Notif) Tab'

However, the 'pointer' tag itself as well as all other elements in the DMSREJ notification would align to WCO IDs found in the 'WCO References (Trader Notif) Tab in the Codelist Document

For AdditionalInformation, the following elements would apply to the DMSREJ notification and would align to WCO IDs found in the 'WCO References (Trader Notif) Tab in the Codelist Document

Pointer (97A) -> DocumentSectionCode (375)

Pointer (97A) -> SequenceNumeric (006)

Pointer (97A) -> Tag ID (381)

| **Element name** | **WCO ID** | DMSREJ |
| --- | --- | --- |
| **AdditionalInformation** | **03A** | X   |
| StatementCode | 226 | X   |
| LimitDateTime | 264 |     |
| StatementTypeCode | 369 | X   |
| **Pointer** | **97A** | X   |
| DocumentSectionCode | 375 | X   |
| SequenceNumeric | 006 | X   |
| TagID | 381 | X   |
| **Amendment** | **06A** |     |
| ChangeReasonCode | 099 |     |
| **Pointer** | **97A** |     |
| DocumentSectionCode | 375 |     |
| SequenceNumeric | 006 |     |
| TagID | 381 |     |

For Amendments the following elements would **not** apply for DMSREJ Notification

Pointer (97A) -> DocumentSectionCode (375)

Pointer (97A) -> SequenceNumeric (006)

Pointer (97A) -> Tag ID (381)

For information on how to amend, please look at section 5.1.8.

#### DMSCTL

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSCTL=05 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/ID | an..35 |     |
| \--Declaration/VersionID | an..9 |     |

#### DMSDOC

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSDOC=06 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-AdditionalInformation** | 0..\* | Used to specify which document(s) should be provided. |
| \--AdditionalInformation/StatementCode | an..17 | Document type code. |
| \--AdditionalInformation/StatementDescription | an..512 | Document identifier. |
| \--AdditionalInformation/LimitDateTime | an..35 | The due date to present the documents. |
| \--AdditionalInformation/StatementTypeCode | an..3 | “ACA” (document type to be presented for document control). |
| **\--Pointer** | 0..\* |     |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| \---Pointer/TagID | an..4 |     |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |

#### DMSRES

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSRES=07 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-AdditionalInformation** | 0..\* | Used to specify the reason for some amendment. |
| \--AdditionalInformation/StatementDescription | an..512 | Textual reason for amendment if it was provided with amendment |
| \--AdditionalInformation/StatementTypeCode | an..3 | “AES” (customs position motivation) from #AdditionalInformationTypes \[4\] |
| **\--Pointer** | 0..\* | Pointer to the Amendment object within the trader notification that the AdditionalInformation relates to. |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| \---Pointer/TagID | an..4 |     |
| **\-Amendment** | 0..\* |     |
| \--Amendment/ChangeReasonCode | an..4 |     |
| **\--Pointer** | 0..\* | Pointer to what was amended in the declaration. |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| \---Pointer/TagID | an..4 |     |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |
| **\-Declaration (Relevant Parts)** | 1..1 | For DMSRES, relevant parts that were amended will be included using the full WCO Declaration structure. |

#### DMSROG

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSROG=08 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-AdditionalInformation** | 0..\* | Used to specify the overall control result, the reason of the customs position, and any observations found during control. |
| \--AdditionalInformation/StatementCode | an..17 | (where statement type code = “AFB”): The overall control result as defined in #OverallControlResultTypes \[4\]<br><br>(where statement type code = “BLF”): The control type for which the observation has been done (as per #ControlTypes \[4\]) |
| \--AdditionalInformation/StatementDescription | an..512 | (where statement type code = “AFB”): The motivation for the customs position.<br><br>(where statement type code = “BLF”): The explanation for the control result. |
| \--AdditionalInformation/LimitDateTime | an..35 | (Statement type “BLF”): If present, indicates the date the control result has become final. |
| \--AdditionalInformation/StatementTypeCode | an..3 | “BLF” (control explanation) or “AFB” (customs position motivation) from #AdditionalInformationTypes \[4\] |
| **\--Pointer** | 0..\* | If the control explanation was in regard to any observations made, the pointer will indicate which element in the declaration the control was on. |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| \---Pointer/TagID | an..4 |     |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |

#### DMSCLE

| **Element** | **Format** | **Description** |     |
| --- | --- | --- |     | --- |
| **Response** | 1..\* |     |     |
| \-Response/IssueDateTime | an..35 |     |     |
| \-Response/FunctionCode | n..2 | DMSCLE=09 |     |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |     |
| **\-AdditionalInformation** | 0..\* | Used to specify the overall control result, the reason of the customs position, and any observations found during control. |     |
| \--AdditionalInformation/StatementCode | an..17 | (where statement type code = “AFB”): The overall control result as defined in #OverallControlResultTypes \[4\]<br><br>(where statement type code = “BLF”): The control type for which the observation has been done (as per #ControlTypes \[4\]) |     |
| \--AdditionalInformation/StatementDescription | an..512 | (where statement type code = “AFB”): The motivation for the customs position.<br><br>(where statement type code = “BLF”): The explanation for the control result. |     |
| \--AdditionalInformation/LimitDateTime | an..35 | (Statement type “BLF”): If present, indicates the date the control result has become final. |     |
| \--AdditionalInformation/StatementTypeCode | an..3 | “BLF” (control explanation) or “AFB” (customs position motivation) from #AdditionalInformationTypes \[4\] |     |
| **\--Pointer** | 0..\* | If the control explanation was in regard to any observations made, the pointer will indicate which element in the declaration the control was on. |     |
| \---Pointer/DocumentSectionCode | an..3 |     |     |
| \---Pointer/SequenceNumeric | n..5 |     |     |
| \---Pointer/TagID | an..4 |     |     |
| **\-Declaration** | 1..1 | Will occur once. |     |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |     |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |     |
| \--Declaration/VersionID | an..9 |     |     |

#### DMSINV

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSINV=10 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-AdditionalInformation** | 0..\* | Used to convey the reason of the customs position. |
| \--AdditionalInformation/StatementCode | an..17 | The reason for invalidation, coded within #CustomsPositionReasonTypes \[4\]. |
| \--AdditionalInformation/StatementDescription | an..512 | The motivation for the customs position. |
| \--AdditionalInformation/StatementTypeCode | an..3 | “AFB” (customs position motivation). |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/CancellationDateTime | an..35 |     |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |

#### DMSREQ

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSREQ=11 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-AdditionalInformation** | 0..\* | Used to convey the reason of the customs position. |
| \--AdditionalInformation/StatementDescription | an..512 | The motivation for the customs position. |
| \--AdditionalInformation/StatementTypeCode | an..3 | “AFB” (customs position motivation). |
| **\-Status** | 0..\* | Used to specify the customs position for the additional message.<br><br>Note: while the schema allows multiple statuses to be returned, a DMSREQ will only ever contain one status |
| \--Status/NameCode | an..3 | Value will be one of:<br><br>39 – Granted<br><br>41 – Denied |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related additional message. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |

#### DMSTAX

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSTAX=13 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-Appeal Office** | 0..1 | May contain the identification of the office of appeal, however, will not be populated currently. |
| \--Appeal Office/ID |     |     |
| **\-Bank** | 0..1 | Contains the bank account details for payments |
| \--Bank/ID | an..17 | The bank account number (e.g. national or IBAN/SEPA code). |
| \--Bank/ReferenceID | an..35 | An identification of the bank (e.g. BIC code). |
| **\-ContactOffice** | 0..\* | May contain the contact details for customs offices, e.g. for the office of appeal. However, will not be populated currently. |
| \--ContactOffice/ID |     |     |
| **\--Communication** |     |     |
| \---Communication/ID |     |     |
| \---Communication/TypeCode |     |     |
| **\-Status** | 0..\* | Used to specify the status of the customs debt.<br><br>Note: while the schema allows multiple statuses to be returned, a DMSTAX will only ever contain one status |
| \--Status/NameCode | an..3 | Value will be one of:<br><br>67 – Indicative customs debt.  <br>115 – Provisional customs debt.  <br>4 – Final customs debt. |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |
| **\--DutyTaxFee** |     | Will only occur for a cash payment |
| **\---Payment** |     | Will occur at most once.<br><br>Note – where the calculation is indicative (before the customs position is determined), no payment information will be provided. |
| \----Payment/PaymentAmount | n..16,2 | Tax assessed minus suspended and transferred amounts for DutyTaxFee. |
| \----Payment/DueDateTime |     |     |
| \----Payment/ReferenceID | an..35 | The payment reference in case of cash payment. |
| \----Payment/TaxAssessedAmount | n..16,2 | Aggregated tax assessed amount in case of cash payment. |
| **\--Goods Shipment** |     |     |
| **\---GovernmentAgencyGoodsItem** |     |     |
| \----GovernmentAgencyGoodsItem/SequenceNumeric | n..5 |     |
| **\----Commodity** |     |     |
| **\-----DutyTaxFee** |     | Used for DutyTaxFees and Securities<br><br>Note: DutyTaxFees representing a security are distinguishable from DutyTaxFees representing actual duty/tax/fees by the fact that PaymentAmount is not provided for securities. |
| \------DutyTaxFee/AdValoremTaxBaseAmount | n..16,2 | This will be included if the DutyTaxFee (tax line) is the result of a "value-based" measure e.g. VAT being calculated as 20% of the value of the goods. From a logical perspective, this base multiplied by the rate will correspond to the tax-assessed amount prior to relief being applied. |
| \------DutyTaxFee/DeductAmount | n..16,2 | Amount of DutyTaxFee relief . |
| \------DutyTaxFee/DutyRegimeCode | an..3 | Contains the preference code actually used for calculation. |
| \------DutyTaxFee/TaxRateNumeric | an..17,2 | Will be a percentage for AdValorem duties (unitCode="P1") or an amount with currency for Specified duties. |
| \------DutyTaxFee/TypeCode | an..3 | DutyTaxFee tax type or Security amount type. Security amount types are coded within #SecurityAmountTypes \[4\]. |
| **\------Payment** |     |     |
| \-------Payment/PaymentAmount | n..16,2 | Tax assessed minus suspended and transferred amounts for DutyTaxFee (empty for Securities). |
| \-------Payment/TaxAssessedAmount | n..16,2 | The calculated DutyTaxFee amount including relief and adjustments or the Security amount. |

##### DMSTAX Response representing outright payable DutyTaxFees

Calculated DutyTaxFees in the DMSTAX response representing outright payable DutyTaxFees have both the TaxAssessedAmount and PaymentAmount elements populated.

**Example Declaration Response**

&lt;DutyTaxFee&gt;  
&lt;AdValoremTaxBaseAmount currencyID="GBP"&gt; 200.00&lt;/AdValoremTaxBaseAmount&gt;  
&lt;DeductAmount currencyID="GBP"&gt;0.00&lt;/DeductAmount&gt;  
&lt;DutyRegimeCode&gt;100&lt;/DutyRegimeCode&gt;  
&lt;TaxRateNumeric&gt;2.00&lt;/TaxRateNumeric&gt;  
&lt;TypeCode&gt;A00&lt;/TypeCode&gt;  
&lt;Payment&gt;  
&lt;TaxAssessedAmount currencyID="GBP"&gt;4.00&lt;/TaxAssessedAmount&gt;  
&lt;PaymentAmount currencyID="GBP"&gt;4.00&lt;/PaymentAmount&gt;  
&lt;/Payment&gt;  
&lt;/DutyTaxFee&gt;

##### DMSTAX Response representing a Security

Calculated DutyTaxFees in the DMSTAX response representing a security are distinguishable from calculated DutyTaxFees representing outright payable DutyTaxFees by the fact that the additional PaymentAmount element (after TaxAssessedAmount element) is **not** included for securities. Also, note that the SecurityAmountType (see next section) is entered in the TypeCode element.

**Example Declaration Response**

A **different** declaration may have a Calculated DutyTaxFees DMSTAX response as follows:

&lt;DutyTaxFee&gt;  
&lt;TypeCode&gt;MDC&lt;/TypeCode&gt;  
&lt;Payment&gt;  
&lt;TaxAssessedAmount currencyID="GBP"&gt; 222.75&lt;/TaxAssessedAmount&gt;  
&lt;/Payment&gt;  
&lt;/DutyTaxFee&gt;

In this Declaration Response example, the TypeCode element signifies that the tax will be applicable against a SecurityAmountType MDC.

##### Provisional Anti-Dumping Duty (PDD)

CDS calculates the duty amount associated with a Provisional Anti-Dumping (PADD) and/or Provisional Countervailing Duty (PCVD) measure, imposed on a commodity code from a specific country of origin. The specific rate that applies will be indicated by the TARIC additional Code entered in DE 6/16 (TRA).  
CDS will then collect this as a security amount for any provisional PADD and/or PCVD. CDS distinguishes this in the finance system as a security payment with a reason provided - Providing ‘PDD’ (Provisional Dumping Duty) as the reason for the security.

The security deposit against PDD is reserved against the declared account based or immediate payment method entered on the declaration and is reserved separately from any outright payments.  
HMRC holds the PDD amount as a security deposit pending the outcome of the decision on the antidumping investigation. The decision will result in either a full/partial refund of the deposit to the trader or bringing the deposit to account.

HMRC ADD Policy will then notify traders that they will need to submit claims to NIDAC/NDRC in order to release any excess security collected when these cases are completed (notices are also provided on the GOV.UK website). HMRC will then bring the security to account or refund the trader any excess security due, via the original method of payment used for the collection of the security.

**Example Calculation**

| Base Amount | Type Code | Tax Rate | Assessed Amount | Payment Amount | Notes |
| --- | --- | --- | --- | --- | --- |
| £224188.21 | A00 | 6.00% | £13451.29 | £13451.29 | Normal 6% duty on 224188.21 |
| £224188.21 | A35 | 9.50% | £0.00 | £0.00 | 9.5% A35 ADD not applied, taken as PDD |
| £237639.50 | B00 | 20.00% | £47527.90 | £0.00 | 224188.21 value + 13451.29 duty = value for VAT of 237639.50 |
| £224188.21 | PDD | 9.50% | £21297.87 |     | the 9.5% PDD due on 224188.21 |
| £21297.87 | PDD | 20.00% | £4259.57 |     | 20% VAT due on the PDD duty amount of 21297.87 |

There are typically 5 DutyTaxFee data elements returned in the DMSTAX:

A00, A35, B00, PDD (B00), PDD (A35)

At first glance, it may look like the A35 tax line is coming out at £0, and the reason is that the provisional duty is being secured rather than taken outright - the PDD amount(s) reflect this below the payable tax lines (though noting that we don't indicate which is for the A35 and which is for the B00).

**Example Declaration Response**

&lt;DutyTaxFee&gt;  
&lt;AdValoremTaxBaseAmount currencyID="GBP"&gt;224188.21&lt;/AdValoremTaxBaseAmount&gt;  
&lt;DeductAmount currencyID="GBP"&gt;0.00&lt;/DeductAmount&gt;  
&lt;DutyRegimeCode&gt;100&lt;/DutyRegimeCode&gt;  
&lt;TaxRateNumeric&gt;6.00&lt;/TaxRateNumeric&gt;  
&lt;TypeCode&gt;A00&lt;/TypeCode&gt;  
&lt;Payment&gt;  
&lt;TaxAssessedAmount currencyID="GBP"&gt;13451.29&lt;/TaxAssessedAmount&gt;  
&lt;PaymentAmount currencyID="GBP"&gt;13451.29&lt;/PaymentAmount&gt;  
&lt;/Payment&gt;  
&lt;/DutyTaxFee&gt;  
&lt;DutyTaxFee&gt;  
&lt;AdValoremTaxBaseAmount currencyID="GBP"&gt;224188.21&lt;/AdValoremTaxBaseAmount&gt;  
&lt;DeductAmount currencyID="GBP"&gt;0.00&lt;/DeductAmount&gt;  
&lt;DutyRegimeCode&gt;100&lt;/DutyRegimeCode&gt;  
&lt;TaxRateNumeric&gt;9.50&lt;/TaxRateNumeric&gt;  
&lt;TypeCode&gt;A35&lt;/TypeCode&gt;  
&lt;Payment&gt;  
&lt;TaxAssessedAmount currencyID="GBP"&gt;0.00&lt;/TaxAssessedAmount&gt;  
&lt;PaymentAmount currencyID="GBP"&gt;0.00&lt;/PaymentAmount&gt;  
&lt;/Payment&gt;  
&lt;/DutyTaxFee&gt;  
&lt;DutyTaxFee&gt;  
&lt;AdValoremTaxBaseAmount currencyID="GBP"&gt;237639.50&lt;/AdValoremTaxBaseAmount&gt;  
&lt;DeductAmount currencyID="GBP"&gt;0.00&lt;/DeductAmount&gt;  
&lt;DutyRegimeCode&gt;100&lt;/DutyRegimeCode&gt;  
&lt;TaxRateNumeric&gt;20.00&lt;/TaxRateNumeric&gt;  
&lt;TypeCode&gt;B00&lt;/TypeCode&gt;  
&lt;Payment&gt;  
&lt;TaxAssessedAmount currencyID="GBP"&gt;47527.90&lt;/TaxAssessedAmount&gt;  
&lt;PaymentAmount currencyID="GBP"&gt;0.00&lt;/PaymentAmount&gt;  
&lt;/Payment&gt;  
&lt;/DutyTaxFee&gt;  
&lt;DutyTaxFee&gt;  
&lt;TypeCode&gt;PDD&lt;/TypeCode&gt;  
&lt;Payment&gt;  
&lt;TaxAssessedAmount currencyID="GBP"&gt;21297.87&lt;/TaxAssessedAmount&gt;  
&lt;/Payment&gt;  
&lt;/DutyTaxFee&gt;  
&lt;DutyTaxFee&gt;  
&lt;TypeCode&gt;PDD&lt;/TypeCode&gt;  
&lt;Payment&gt;  
&lt;TaxAssessedAmount currencyID="GBP"&gt;4259.57&lt;/TaxAssessedAmount&gt;  
&lt;/Payment&gt;  
&lt;/DutyTaxFee&gt;

##### List of SecurityAmountTypes

The following table (taken from the System Defined Codes in CodeLists) shows a list of the 18 SecurityAmountTypes, one of which gets incorporated in the Calculated DutyTaxFees in the DMSTAX response representing a security , not the outright payable DutyTaxFees, when a security is calculated.

#### DMSCPI

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSCPI=14 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-Bank** | 0..1 | May contain the bankaccount details. |
| \--Bank/ID |     | The bankaccount number (e.g. national or IBAN/SEPA code). |
| \--Bank/ReferenceID |     | An identification of the bank (e.g. BIC code). |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |
| **\--DutyTaxFee** |     | Provided per coverage means. |
| **\---Payment** |     | Will occur at most once. |
| \----Payment/PaymentAmount | n..16,2 | The requested pay-up/increase amount. |
| \----Payment/DueDateTime |     |     |
| \----Payment/ReferenceID | an..35 | The payment reference in case of cash payment |
| **\----Obligation Guarantee** |     |     |
| \-----ObligationGuarantee/ReferenceID | an..35 | The account ID relevant to the guarantee increase instruction. E.g. DAN. |

#### DMSCPR

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSCPR=15 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-Bank** | 0..1 | May contain the bankaccount details. |
| \--Bank/ID |     | The bankaccount number (e.g. national or IBAN/SEPA code). |
| \--Bank/ReferenceID |     | An identification of the bank (e.g. BIC code). |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |
| **\--DutyTaxFee** |     | Provided per coverage means. |
| **\---Payment** |     | Will occur at most once. |
| \----Payment/PaymentAmount | n..16,2 | The requested pay-up/increase amount. |
| \----Payment/DueDateTime |     |     |
| \----Payment/ReferenceID | an..35 | The payment reference in case of cash payment. |
| **\----Obligation Guarantee** |     |     |
| \-----ObligationGuarantee/ReferenceID | an..35 | The account ID relevant to the guarantee increase instruction. E.g. DAN. |

#### DMSEOG

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSEOG=16 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-AdditionalInformation** | 0..\* | Used to specify the declared/actual office of exit. |
| \--AdditionalInformation/StatementCode | an..17 |     |
| \--AdditionalInformation/StatementDescription | an..512 | The actual office of exit. |
| \--AdditionalInformation/StatementTypeCode | an..3 | “CEX” (clearance instructions for export). |
| **\-Status** |     | Used to specify the exit date and overall control result at exit. |
| \--Status/EffectiveDateTime | an..35 |     |
| \--Status/NameCode | an..3 | Values from code list OverallControlResultTypes |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 | .   |

The DMSEOG is triggered by the Office of Exit, and unfortunately we do not have any emulator that will trigger this message in Trade Test or in Trader Dress Rehearsal. It is therefore recommended that, when testing in the TT and TDR environments, SWDs use the EDL ACK as confirmation that their goods have departed.

#### DMSEXT

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSEXT=17 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |

#### DMSGER

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSGER=18 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-AdditionalInformation** | 0..\* | Used to specify a due date. |
| \--AdditionalInformation/LimitDateTime | an..17 | The due date for the goods to be exited. |
| \--AdditionalInformation/StatementTypeCode | an..3 | “CEX” (clearance instructions for export). |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |

#### DMSALV

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/IssueDateTime | an..35 |     |
| \-Response/FunctionCode | n..2 | DMSALV=50 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| **\-AdditionalInformation** | 0..\* | Contains the ALVS decision |
| \--AdditionalInformation/StatementCode | an..17 | Decision code corresponding to the #DecisionCodes codelist |
| \--AdditionalInformation/StatementDescription | an..512 | Free text description of the reason for the decision |
| \--AdditionalInformation/StatementTypeCode | an..3 | “ALV” |
| **\-Error** | 0..\* | Used to indicate that there is a response from Defra that blocks clearance of the declaration |
| \--Error/ValidationCode | an..8 |     |
| \--Error/Description |     |     |
| **\--Pointer** | 0..\* | Points to the goods item that the decision relates to |
| \---Pointer/DocumentSectionCode | an..3 |     |
| \---Pointer/SequenceNumeric | n..5 |     |
| \---Pointer/TagID | an..4 |     |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | CDS-generated reference for the related declaration (the MRN). |
| \--Declaration/VersionID | an..9 |     |

The DMSQRY and DMSALV notifications do not look like a DMS trader notification as expected. There is no difference functionally, because the traders are still successfully receiving the trader notifications, but there are cosmetic differences. This means that a user may think there is an issue with DMS because the notifications they receive do not look like a DMS trader notification. The expectation is that CDS software will need to be able to ingest and report the DMSQRY and DMSALV notifications.

#### DMSQRY

| **Element** | **Format** | **Description** |
| --- | --- | --- |
| **Response** | 1..\* |     |
| \-Response/FunctionCode | n..2 | DMSQRY=51 |
| \-Response/FunctionalReferenceID | an..35 | Functional reference of this notification. |
| \-Response/IssueDateTime | an..35 |     |
| **\-AdditionalInformation** | 0..\* |     |
| \--AdditionalInformation/StatementCode | an..3 | “Q01” |
| \--AdditionalInformation/StatementDescription | an..512 | Static text advising there is a message awaiting action, accessible via the Secure File Upload service. NB: this notification message does not contain the caseworker message, merely advising a message has been sent. |
| \--AdditionalInformation/StatementTypeCode | an..3 | “QRY” |
| **\-Declaration** | 1..1 | Will occur once. |
| \--Declaration/FunctionalReferenceID | an..35 | Submitter reference for the related declaration. |
| \--Declaration/ID | an..35 | DMS generated reference for the related declaration. |
| \--Declaration/VersionID | an..9 |     |

Please note Specifically, response messages to import declarations such as DMSACC and DMSTAX have XML namespace definitions in the ‘Response’ and ‘DateTimeString’ elements. In contrast, the new DMSQRY message has 3 XML namespace definitions in the MetaData element but not in the ‘Response’ element.

## Pointers and Error Codes

### Error Codes

See \[4\] for the description of the error codes that are currently within CDS scope and the category they fall into.

CDS recognises a number of different categories for the error codes which drives internal CDS behaviour but are also relevant for external users of the service. The category is not sent with the validationResultType, so therefore it is for the submitting system to determine the action to take on the response.

| **Category** | **Description** |
| --- | --- |
| RejectingValidationResults | The error code indicates a reason why the declaration has been rejected. |
| PotentiallyRejectingValidationResults | Could be a rejecting result dependent on the Tariff Action Type |
| CredibilityValidationResults | A FEC rule was hit which could indicate a need to amend the declaration. |
| RevalidationRequiringValidationResult | The declaration has not yet been linked with the inventory, and therefore is held in the validation phase until the linking is positive. |

Error codes can follow multiple patterns depending on the type of validation result that is generated. Each pattern will contain at least an error code and a pointer, some may contain additional information.

| **Validation Result Type** | **Expected Response** | **Example error** |
| --- | --- | --- |
| Single field failure on a declared value | A single error code with a pointer to the specific offending field. | Submitter reference is not unique |
| Cross-field failure where both fields are declared | Two error codes with two pointers to the specific fields. | Authorisation holder is declared but not the relevant authorisation |
| Tariff validation failure | An error code, with pointer, and an additional information object with the measure conditions. | Licence requirement not met |
| FEC failure | A FEC error code, with pointer to the goods item, and an additional information object containing the actual error. | Mass to value ratio too high |
| Inventory Linking failure | An error code, with pointer, and an optional additional information object with the ICS value. | Inventory not matched |

### Pointers

#### Description

CDS uses the WCO pointer mechanism to describe the specific data element to which the information relates to. This is used in a number of areas through the system. For example, to identify where the issues occurred in the declaration when it is rejected. As such, to identify the specific problem to resolve, the submitting system will need to interpret the DMSREJ notification to understand the validationResult code, but also the data element that the code relates to. There are multiple areas within CDS where the pointer mechanism is used:

- In Trader Notifications to identify reasons for rejection, or reasons for control;
- Used with the AdditionalInformation object to indicate which element it is related to in the declaration.

The pointer itself is made up of three distinct elements:

- DocumentSectionCode, which is used to identify a specific WCO class (e.g. 42A for Declaration)
- SequenceNumeric, used to identify a particular instance of the specified class. This can either be an explicit sequence number, or implicit based on the order of the elements in the relevant message.
- TagID, the specific attribute in the identified class.

When implicit sequence numbers are used, it is required that in the augmentation message placeholders are used for those classes that are part of the sequence. If this is not done, it is not possible to relate the classes in case of multiple updates in the same sequence. This is demonstrated in the example in section Complex Amendment Example 2.

Some simple examples of the pointer system within a trader notification are described in the next sections. The pointer system is also used within other messages such as amendment requests, and some more complex examples are described in section Complex pointer examples.

Note that where a pointer is used to describe an element that does not exist within the declaration submitted, there will be no SequenceNumeric associated with the pointer. An example would be where a document should have been declared at goods item level due to an element declared at header level but has not been. There is no document to point to, as it is not declared and it could be on any of the goods items submitted, therefore the system cannot determine a correct sequence number. This applies even if there is only one goods item on the declaration, and an example of this is demonstrated in section Simple Error Example 2.

A SequenceNumeric will never show for a singular non-repeating element (e.g. Importer) because it would be redundant.

If a SequenceNumeric is not provided for a repeating element (e.g. AdditionalDocument), this indicates that there is absent data from the declaration, and there is no specific instance of the repeating element to point to.

#### Simple Error Example 1

The below example shows a uniqueness error with the submitterReference (EUCDM 2/5 LRN) sent within the DMSREJ notification.

XML Example

&lt;\_2_1:Error&gt;

&lt;\_2_1:ValidationCode&gt;CDS12003&lt;/\_2_1:ValidationCode&gt;

&lt;\_2_1:Pointer&gt;

&lt;\_2_1:DocumentSectionCode&gt;42A&lt;/\_2_1:DocumentSectionCode&gt;

&lt;\_2_1:TagID&gt;D026&lt;/\_2_1:TagID&gt;

&lt;/\_2_1:Pointer&gt;

&lt;/\_2_1:Error&gt;

Here the declaration is referred to in the DocumentSectionCode (42A), with the specific WCO tag for the submitterReference being used to point to that specific element (D026).

#### Simple Error Example 2

The below example shows a relation error after an AuthorizationHolder (D.E. 3/39) was declared without declaring the pertinent authorisation (D.E. 2/3).

XML Example

&lt;\_2_1:Error&gt;  
&lt;\_2_1:ValidationCode&gt;CDS12056&lt;/\_2_1:ValidationCode&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:DocumentSectionCode&gt;42A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:DocumentSectionCode&gt;67A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:DocumentSectionCode&gt;68A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:DocumentSectionCode&gt;02A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;\_2_1:TagID&gt;D006&lt;/\_2_1:TagID&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt; /\_2_1:Error&gt;

Here the pointer objects represent the following XPath: /Declaration/ConsignmentShipment/GovernmentAgencyGoodsItem/AdditionalDocument/TypeCode. As the AuthorizationHolder is declared at header-level, and the system expects the authorisation to be declared within an AdditionalDocument in at least one of the goods items in the declaration, this error can’t point to a specific AdditionalDocument element on any specific GovernmentAgencyGoodsItem element within the declaration, and hence doesn’t include any SequenceNumeric elements. Further information is provided in section Description.

### Additional Information

Additional information objects can be used to supplement error codes with further detail on what has failed. These are optional and are dependent on the error code.

They are used for Tariff and Inventory Linking validation results. Multiple additional information fields can be used with regards to Tariff errors.

For example, if a certain certificate has not been declared against a commodity – the measure could relate to multiple possible certificates. Each of these possible options are returned in separate additional information blocks which would relate to the one error.

#### Simple AdditionalInformation Example 1

The below example shows a FEC warning being returned, e.g. in a DMSRCV trader notification.

XML Example

&lt;\_2_1:AdditionalInformation&gt;  
&lt;\_2_1:StatementCode&gt;smartErrorMsg&lt;/\_2_1:StatementCode&gt;  
&lt;\_2_1:StatementDescription&gt;Value per kilo appears too high for this commodity&lt;/\_2_1:StatementDescription&gt;  
&lt;\_2_1:StatementTypeCode&gt;1&lt;/\_2_1:StatementTypeCode&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:SequenceNumeric&gt;1&lt;/\_2_1:SequenceNumeric&gt;  
&lt;\_2_1:DocumentSectionCode&gt;07B&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:SequenceNumeric&gt;1&lt;/\_2_1:SequenceNumeric&gt;  
&lt;\_2_1:DocumentSectionCode&gt;53A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;/\_2_1:AdditionalInformation&gt;  
&lt;\_2_1:Error&gt;  
&lt;\_2_1:ValidationCode&gt;CDS13000&lt;/\_2_1:ValidationCode&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:DocumentSectionCode&gt;42A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:DocumentSectionCode&gt;67A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;\_2_1:Pointer&gt;  
&lt;\_2_1:SequenceNumeric&gt;1&lt;/\_2_1:SequenceNumeric&gt;  
&lt;\_2_1:DocumentSectionCode&gt;68A&lt;/\_2_1:DocumentSectionCode&gt;  
&lt;/\_2_1:Pointer&gt;  
&lt;/\_2_1:Error&gt;

Here there is an Error element, with a validation code that indicates that a FEC warning result has been raised, pointing to the relevant goods item (in this case, the first goods items). In addition to the goods item pointer, there will be an AdditionalInformation element provided that includes more detail. The Error indicates that something is wrong in the declaration, and the AdditionalInformation indicates what it is that is wrong.

Each Error can have multiple AdditionalInformation elements relating to it, and as they are both defined as siblings in the schema, the WCO Pointer mechanism is used to link one to another. While the pointers contained in the Error element point back to declaration elements, the pointers contained in the AdditionalInformation element point to the Error element within the trader notification that it relates to.

In the example above, the pointers in the AdditionalInformation element resolve to an XPath that identifies the first Error element in the first Response element, which means that the statement of “Value per kilo appears too high for this commodity” relates to the CDS13000 error, which relates to the first goods item on the declaration.

### Tariff Measure Validation Failures

Tariff Measures can either have a condition where a document needs to be provided, or a condition where a specific element or ratio needs to be in a certain range.

CDS will generate an individual error block for each condition not fulfilled. The error code values will either be CDS40045 (missed document) or CDS40066 (measure condition not fulfilled). These error blocks can have further information associated to them contained in the AdditionalInformation object.

The scope of information provided in the AdditionalInformation objects includes (StatementTypeCode in brackets):

- The specific value needs to be in a certain range (Type 1)
- The ratio of values that can be provided for a certain measure (Type 3)
- The document type that needs to be declared with the commodity (Type 5)
- Any action codes associated with the condition (Type 6)
- The measure condition that has not been met (Type 7)
- Whether all documents need to be declared or just one (Type 9)

Each additional information object will contain the following elements:

- **StatementDescription** will contain a description of the error and/or detail on the ratio and the range it needs to meet. This field will also, where applicable, contain the measure type and the commodity code, the condition code and the action code.
- **StatementCode** will contain the measure condition code, action code and document code depending on the StatementTypeCode. CDS will not be returning any values or quantities in this field so the code may not be returned in all circumstances.
- **StatementTypeCode** will contain the type of code that has been returned in StatementCode.

Examples of the additional information that can accompany a Tariff validation failure:

StatementDescription = "The declared price must be equal or greater than " & **Quantity Condition** & **Currency Condition** & "/" & **Measurement Unit**

StatementTypeCode = **1**

StatementDescription = "The ratio declared value/supplementary unit should be higher than " & **Quantity Condition & Currency Condition** & "/" & **Measurement Unit for Condition**

StatementTypeCode = **3**

StatementDescription = "The measure type ' " & **Measure Type ID** & " ' on commodity code ' " & **Commodity Code** & " ' requires document"

StatementTypeCode = 5

StatementCode = **Certificate Code**

StatementDescription = "The measure condition '" & **Condition Code** & " ' has the action code"

StatementTypeCode = 6

StatementCode = **Action Code**

StatementDescription = "The measure type ' " & **Measure Type ID** & " ' on commodity code ' " & **Commodity Code** & " ' has a measure condition"

StatementTypeCode = 7

StatementCode = **Condition Code**

StatementDescription = Either of those documents need to be supplied

StatementTypeCode = 9

StatementCode = All

### List of WCO Data Element Tags

See CDS Codelists and WCO Reference document \[4\].


# Interface Definitions

## Function Codes and Meta-Data

The WCO model specifies a single message structure that can be used in multiple ways, driven by the function code specified in the message. All declaration submissions will have a message function code of ‘9’, whereas the additional message submissions have a code of ’13’.

Where ‘xx’ is referenced, that will refer to the trade movement type (IM, EX, or CO)

| **Submission** | **Message Function Code** | **Declaration Type Code** |
| --- | --- | --- |
| Normal Declaration | 9   | xxA |
| Simplified ‘B’ Declaration | 9   | xxB |
| Simplified ‘C’ Declaration | 9   | xxC |
| Advance Declaration | 9   | xxD |
| Advanced Simplified Declaration | 9   | xxE |
| Advanced Simplified Declaration | 9   | xxF |
| EIDR Supplementary Declaration | 9   | xxZ |
| Supplementary Declaration after B/C | 9 (on the assumption these are declarations rather than additional messages linked to the simplified) | xxX OR xxY |
| Goods Presentation | 13  | GPR |
| Request for Invalidation | 13  | INV |
| Request for Amendment | 13  | COR |

## Data Element Matrix

See CDS Declaration Technical Completion Matrix \[5\]

Note

The Technical Completion Matrix will be updated as the validation rules per declaration type are elaborated.

# Schemas

As specified in the latest version of the API Specification \[1\].




**OFFICIAL**

© HM Revenue & Customs 2022.

All rights reserved.
