# Manual Deployment
 - Create a DynamoDB table where:
   - The key of the schema will be the device code
   - One global secondary index will index the user code
   - One global secondary index will index the state of the OAuth2 Authorization Code grant flow request/response
 - Create a new Cognito User Pool
   - Create an Authentication Domain
 - Create a new Lambda function (NodeJS based, tested with nodejs10.x)
   - Deploy the files in this repository
 - Associate an IAM Execution role that allows:
   - Basic Lambda Execution permissions
   - Access to the DynamoDB table for Read, Update, and Delete operations
   - Access to the Cognito User Pool as Power User
 - Create thirteen environment variable:
   - `CODE_EXPIRATION` that represents the lifetime in seconds of the codes generated
   - `CODE_VERIFICATION_URI` that references the URI where the end user should authorize or deny the authorization request using the user code
   - `CUP_DOMAIN` that references the Cognito User Pool prefixed domain name
   - `CUP_ID` that references the Cognito User Pool ID
   - `CUP_REGION` that references the Cognito User Pool region
   - `DEVICE_CODE_FORMAT` that represents the format for the device code (for example: `#aA` where `#` represents numbers, `a` lowercase letters, `A` uppercase letters, and `!` special characters)
   - `DEVICE_CODE_LENGTH` that represents the device code length
   - `DYNAMODB_AUTHZ_STATE_INDEX` that references the name of the global secondary index will index the state of the OAuth2 Authorization Code grant flow request/response in the DynamoDB table
   - `DYNAMODB_TABLE` that references the DynamoDB table
   - `DYNAMODB_USERCODE_INDEX` that references the name of the global secondary index will index the user code in the DynamoDB table
   - `POLLING_INTERVAL` that represents the minimum time in seconds between two polling from the client application
   - `USER_CODE_FORMAT` that represents the format for the user code (for example: `#aA` where `#` represents numbers, `a` lowercase letters, `b` lowercase letters without vowels,`A` uppercase letters, `B` uppercase letters without vowels, and `!` special characters)
   - `USER_CODE_LENGTH` that represents the user code length
   - `RESULT_TOKEN_SET` that represents the structure of the Token Set to be returned to the Device. String can only include `ID`,`ACCESS`, and `REFRESH` values separated with `+`
 - Create one ALB instance
 - Create or import in ACM one certificate and its private key that can be used to protect the HTTPS listener of the ALB instance
   - Configure your DNS to ensure a proper routing to the DNS name that is part of the certificate to the ALB instance
   - This DNS name should be in line with what configured in the Lambda function's `CODE_VERIFICATION_URI` environment variable
 - Create two client app credetials in Cognito User Pool with Secret:
   - One for an ALB instance with a callback URL as described in [ALB documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html)
   - One for the client application with a call URL pointing to the same FQDN as in he Lambda function's `CODE_VERIFICATION_URI` environment variable but on the `/callback` endpoint
 - Create Target Group in EC2 that will target the Lambda function created previously
 - Configure the ALB instance with the following listeners:
   - Listener on Port `80` and protocol `HTTP` doing a default redicrect rule to the HTTPS/443 endpoint
   - Listener on Port `443` and protocol `HTTPS` using the certificate created or imported into ACM previously and with:
     - A default fixed `503` response rule
     - A number `1` rule matching `Path` of `/device`:
       - Authenticating with the Cognito User Pool created previously and using the appropriate Client Application credentials
       - Forward to the Target Group created previously
     - A number `2` rule matching `Path` of `/token` or `/callback`:
       - Forward to the Target Group created previously