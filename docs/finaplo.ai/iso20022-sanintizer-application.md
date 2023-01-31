[comment]: # (Use https://www.markdowntopdf.com/ to convert it to pdf)

<a id="logo" href="https://www.paymentcomponents.com" title="Payment Components" target="_blank">
    <img loading="lazy" src="https://i.postimg.cc/yN5TNy29/LOGO-HORIZONTAL2.png" alt="Payment Components">
</a>

# ISO20022 Sanitizer Application Documentation

## Overview
This application reads from local directories ISO20022 xml messages and sanitizes them by removing sensitive data.  
You can either remove elements or hash the values.  
In addition to sanitization, the application splits files that contains many messages or wrappers into simple messages starting with `Document`.  
For example, in `EBA SEPA`, an `ICF` may contains both a `pacs.008.001.02` and an `pacs.004.001.02`. The application with remove `EBA` wrappers 
and will generate two xml files, one for the `pacs.008` and one for the `pacs.004`.

## Prerequisites
- Java 8 or higher
- An active AWS account in case you need to upload the output to AWS S3
- Messages organized in folders `{scheme}/{direction}` where `{scheme}` could be `CBPR`, `SEPA` etc. and `{direction}` could be `in` or `out`

## Running the Application
1. Set the environment variables. See [here](#environment-variables).
2. Run the executable jar:
```
java -jar iso20022-sanitizer-app-1.0.0.jar
```
3. Explore outputs. See [here](#execution-outputs).

## Execution outputs
Once the execution is finished, you will end up with the following assets:
- An execution folder which contains the messages organized in this structure:
```
execution_{timestamp}
       |__sepa
       |     |__in
       |     |   |__pacs.008.001.02
       |     |   |__pacs.004.001.02
       |     |__out
       |     |   |__pacs.008.001.02
       |     |   |__pacs.004.001.02
       |
       |__cbpr
       |    |__in
       |    |   |__pacs.008.001.08
       |    |   |__pacs.004.001.09
       |    |__out
       |    |    |__pacs.008.001.08
       |    |    |__pacs.004.001.09
```
- Inside the execution folder you will also find a `hashed_values.csv` file which contains the hashed value and the real value.  
Columns are `hashed_value,original_value` and this file should not be shared. Hashed value starts with `HS_`.
- A zip file of the above message structure excluding the `hashed_values.csv`.
- Application logs.

## Sanitization Paths Properties File

### Default properties
This application is preconfigured with some default sanitization paths for specific messages.  
They are preloaded in classpath. You can view them by extracting the jar file and browse: `/BOOT-INF/classes/message_properties/`  
The general rule in default sanitization is:
- For every `PartIdentificationXXX`, remove `Name`, all from `Postal Address` except `Country`, `Identification` and `Contact Details`.
- For `Tax Remittance` elements, remove `Creditor`, `Debtor` and `Ultimate Debtor`.
- For every `CashAccountXXX`, hash `IBAN` and `Identification/Other/Identification`.

### File Name
As you can see, each message name id has its own property file and the filename should be `{messagenameid.properties}`.  
For example, a valid filename is `pacs.008.001.08.properties` while `pacs.008.properties` is invalid.  
If you do not specify such a file for a message the application will not sanitize it.  

### File Structure
There are 2 types of sanitization. You can either `REMOVE` an element or `HASH` the content of an element.

If we open a file, we will see the below structure:
```properties
paths.to.hash=\
  CdtTrfTxInf/DbtrAcct/Id/IBAN,\
  CdtTrfTxInf/CdtrAcct/Id/IBAN

paths.to.remove=\
  CdtTrfTxInf/Dbtr/Nm,\
  CdtTrfTxInf/Cdtr/Nm
```
This is an example of a `pacs.008.001.08.properties` file.  
As you can see we can declare 2 categories: `paths.to.hash` and `paths.to.remove`. Values should be comma separated and should start one level after Message Payload.  
For this reason we start from `CdtTrfTxInf/` instead of `/Document/FIToFICstmrCdtTrf/CdtTrfTxInf/`.  

In case of `REMOVE`, it is not necessary to reach the leaf element. Sanitizer will remove the element that the children.  
Also, some xpath expression are allowed. For example, if you want to remove all `PstlAdr/TwnNm`, instead of `CdtTrfTxInf/Dbtr/PstlAdr/TwnNm` and `CdtTrfTxInf/Cdtr/PstlAdr/TwnNm`, 
you can write `CdtTrfTxInf//PstlAdr/TwnNm`. We DO NOT RECOMMEND using such expressions since you may end up removing elements that you want, such as the addresses of agents.

## Environment Variables
- `ISO20022_DIRECTORY_{SCHEME}_{DIRECTION}` (default: `empty`): Specify the directory where messages of a specific Message Scheme (SEPA, CBPR) and direction (IN, OUT) exist.
- `EXTERNAL_SANITIZER_PROPERTIES_PATH` (default: `empty`): In case you do not want to use the default sanitization paths, you can declare your own properties. Full path is needed. See [how](#sanitization-paths-properties-file).
- `ALLOWED_MESSAGE_PAYLOADS` (default: `"FIToFICstmrCdtTrf", "PmtRtr", "FIToFIPmtStsRpt", "FIToFIPmtStsRptS2", "FIToFIPmtCxlReq", "RsltnOfInvest"`): Declare the message payloads that the application will process.
- `OUTPUT_FOLDER_PATH` (default: `./output`): The directory where the zip file and the execution folder will be stored.
- `S3_BUCKET_NAME` (default: `empty`): Normally starts with `pc14-financial-messages-`
- `S3_REGION` (default: `empty`): AWS region in form `eu-west-1`
- `S3_ACCESS_KEY` (default: `empty`): An AWS access key.
- `S3_SECRET_KEY` (default: `empty`): An AWS secret key.
- `LOGGING_LEVEL` (default: `INFO`): Logging level.

Note: These variables should be set before running the application.


Example (for Windows use `set` and for Linux use `export`):
```
ISO20022_DIRECTORY_SEPA_IN=C:\Users\sepa_in
ISO20022_DIRECTORY_SEPA_OUT=C:\Users\sepa_out
ISO20022_DIRECTORY_CBPR_IN=C:\Users\cbpr_in
ISO20022_DIRECTORY_CBPR_OUT=C:\Users\cbpr_out
S3_BUCKET_NAME=pc14-financial-messages-37287373111
S3_REGION=eu-west-1
S3_ACCESS_KEY=your_access_key
S3_SECRET_KEY=your_secret_key
```

## Additional Notes
If any of the S3 related variables is not present, the zip file will not be uploaded to S3.