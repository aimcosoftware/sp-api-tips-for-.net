# Amazon SP API tips for .Net
Tips for connecting to Amazon Selling Partner API from .Net. Some of the things we have learned while setting up SP API

This isn't an exhaustive guide, it's just the basic steps for signing and making calls, that we found it hard to extract from the official docs.

#### Basic Steps
1. [Register as a developer](https://developer-docs.amazon.com/sp-api/docs/registering-as-a-developer) on Amazon
2. Create Amazon App and [setup IAM user on AWS](https://developer-docs.amazon.com/sp-api/docs/creating-and-configuring-iam-policies-and-entities)
3. Use Amazon App credentials to get LWA or RDT Access Token
4. Use IAM credentials to get STS token used to sign SP API requests 
5. Use Access Token and STS credentials for signing, to make requests to SP API 

#### We have only used pseudo code here, except where a specific library is required

## 1. Get the LWA AccessToken
### Required to get STS token for signing SP API requests

Use your ClientId and ClientSecret from your Amazon App and your client's Refresh Token to get an Access Token
```
' Get LWA token using your Amazon App credentials
Sub GetLWAToken()
	' Create JSON body (we use our own JSON library)
	Json.OpenBrace()
	Json.AddPair("grant_type", "refresh_token")
	Json.AddPair("client_id", "amzn1.application-oa2-client.xxxxxxx")
	Json.AddPair("client_secret", "xxxxxxx")
	Json.AddPair("refresh_token", "Atzr|xxxxxxx")
	Json.CloseBrace()

	' Make the request (we use our own library inherited from WebClient)
	Req.AddHeader("content:application/json")
	Req.AddHeader("content-type:application/json")
	Res = Req.PostData("https://api.amazon.com/auth/o2/token", Json.ToString)

	' Retrieve the Access Token (valid for 1 hour)
	AccessToken = JsonValue(Res, "access_token")
End Sub
```
## 2. Get the AWS STS token from your IAM User
### Required to make SP API calls
Use IAM user role credentials to get a security token and signing credentials
```
' Get STS credentials using your IAM user credentials
Sub GetSTSCredentials()
	TimeStamp = UTC.ToString("yyyyMMddTHHmmssZ")
	DateStamp = UTC.ToString("yyyyMMdd")

	' Uri encoded form body
	Body = $"Action=AssumeRole&Version=2011-06-15&RoleArn=arn%3Aaws%3Aiam%3A%3Axxxxxxxxxxxx%3Arole%2FSellingPartnerAPIRole&RoleSessionName={Now.ToUnixTime}"

	' Uri details to hash for signing, region is us-east-1, eu-west-1, etc
	Post = "POST" & vbLf
	Post &= "/" & vbLf & vbLf

	Post &= $"host:sts.{Region}.amazonaws.com" & vbLf
	Post &= $"x-amz-date:{TimeStamp}" & vbLf & vbLf

	Post &= "host;x-amz-date" & vbLf
	Post &= Hash(Body)

	' AWS4 string to sign
	Sign = "AWS4-HMAC-SHA256" & vbLf
	Sign &= TimeStamp & vbLf
	Sign &= $"{DateStamp}/{Region}/sts/aws4_request" & vbLf
	Sign &= Hash(Post)

	SecretKey = "AWS IAM user secret key"
	AccessKey = "AWS IAM user access key"

	' Authorization header (Signature function is below)
	Req.AddHeader($"Authorization:AWS4-HMAC-SHA256 Credential={AccessKey}/{DateStamp}/{Region}/sts/aws4_request, SignedHeaders=host;x-amz-date, Signature={Signature(Sign, "sts")}")

	' Add other headers and make request
	Req.AddHeader("content-type:application/x-www-form-urlencoded")
	Req.AddHeader($"host:sts.{Region}.amazonaws.com")
	Req.AddHeader($"x-amz-date:{TimeStamp}")
	Dim Res = Req.PostData($"https://sts.{Region}.amazonaws.com", Body)

	' Credentials to sign SP API calls (valid for 1 hour)
	AccessKey = XMLValue(Res, "AccessKeyId")
	SecretKey = XMLValue(Res, "SecretAccessKey")
	SecureToken = XMLValue(Res, "SessionToken")
End Sub
```
## 3. Signing and hashing support
```
' Signature to authorize SP API and STS requests
Function Signature(StringToSign As String, Service As String) As String
	kSecret = Encoding.UTF8.GetBytes($"AWS4{SecretKey}")
	kDate = Hmac(UTC.ToString("yyyyMMdd"), kSecret)
	kRegion = Hmac(Region, kDate)
	kService = Hmac(Service, kRegion)
	kSigning = Hmac("aws4_request", kService)
	Return Hmac(StringToSign, kSigning).ToHex
End Function

Function Hmac(Data As String, Key As Byte()) As Byte()
	' Use this specific class from System.Security.Cryptography
	Using Hma = KeyedHashAlgorithm.Create("HmacSHA256")
		Hma.Key = Key
		Return Hma.ComputeHash(Encoding.UTF8.GetBytes(Data))
	End Using
End Function

Function Hash(Data As String) As String
	Using Sha = HashAlgorithm.Create("SHA256")
		Return Sha.ComputeHash(Encoding.UTF8.GetBytes(Data)).ToHex
	End Using
End Function
```

## 4. Make SP API request
```
' Request using the STS credentials and LWA token
Function AmazonRequest(API As String, Data As String) As String
	UserAgent = "My App/1.0(Language=.Net/4.5.2;Platform=Windows/10)"
	TimeStamp = UTC.ToString("yyyyMMddTHHmmssZ")
	DateStamp = UTC.ToString("yyyyMMdd")
	Host = HostName(EndPoint)

	' Uri to hash for signature
	Post = "POST" & vbLf
	Post &= API & vbLf & vbLf
	
	Post &= "accept:application/json" & vbLf
	Post &= "content-type:application/json" & vbLf
	Post &= $"host:{Host}" & vbLf
	Post &= $"user-agent:{UserAgent}" & vbLf & vbLf
	
	Post &= "accept;content-type;host;user-agent" & vbLf
	Post &= Hash(Data)
	
	' String to sign with STS credentials
	Sign = "AWS4-HMAC-SHA256" & vbLf
	Sign &= TimeStamp & vbLf
	Sign &= $"{DateStamp}/{Region}/execute-api/aws4_request" & vbLf
	Sign &= Hash(Post)

	' Add request headers
	Req.AddHeader($"Authorization: AWS4-HMAC-SHA256 Credential={AccessKey}/{DateStamp}/{Region}/execute-api/aws4_request, SignedHeaders=accept;content-type;host;user-agent, Signature={Signature(Sign, "execute-api")}")
	Req.AddHeader("accept:application/json")
	Req.AddHeader("content-type:application/json")
	Req.AddHeader($"host:{Host}")
	Req.AddHeader($"user-agent:{UserAgent}")
	Req.AddHeader($"x-amz-security-token:{SecureToken}")
	Req.AddHeader($"x-amz-access-token:{AccessToken}")
	Req.AddHeader($"x-amz-date:{TimeStamp}")
	Return Req.PostData($"{EndPoint}{API}", Data)
End Function
```

## .5 Putting it all together to get a MFN label
```
' Get your App, IAM, client and region details here

' Get LWA token using your Amazon App credentials
GetLWAToken()
' Get STS credentials using your IAM user credentials
GetSTSCredentials()

' Create some shipping labels
For each Item in MySalesData
	' Create JSON request (we use our own JSON builder for this)
	JSON.OpenBrace()
	JSON.AddPair("ShippingServiceId", Item.ServiceId)
	JSON.OpenBrace("ShipmentRequestDetails")
	...
	JSON.CloseBrace()
	' Make the call using the STS credentials and LWA token
	Label = AmazonRequest("/mfn/v0/shipments", JSON.ToString)
Next
```
