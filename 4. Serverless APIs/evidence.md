
# Lab 03 Evidence: Serverless APIs

## 1. API Invoke URL
https://a8ykiid5sd.execute-api.us-east-1.amazonaws.com/

---

## 2. Public Endpoint Response
**Command executed:**
`curl.exe -i https://a8ykiid5sd.execute-api.us-east-1.amazonaws.com/health`

**Response Evidence:**
![Screenshot of Public Endpoint Response](./public_endpoint.png)

---

## 3. Protected Endpoint (No Token) Response
**Command executed:**
`curl.exe -i https://a8ykiid5sd.execute-api.us-east-1.amazonaws.com/protected`

**Response Evidence:**
![Screenshot of Expected 401 Failure](./protected_endpoint.png)

---

## 4. Protected Endpoint (With Token) Response
**Command executed:**
`curl.exe -i -H "Authorization: Bearer [YOUR_TOKEN]" https://a8ykiid5sd.execute-api.us-east-1.amazonaws.com/protected`

**Response Evidence:**
![Screenshot of Successful Authenticated Response](./protected_endpoint_with_jwt.png)

---

## 5. CloudWatch Log Request ID
**Log Evidence:**
![Screenshot of CloudWatch Log showing RequestId](./cloud_watch_request_id.png)