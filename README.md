## `app-controller`

A backend service that interacts with the underlying Kubernetes cluster using the Knative serving components.

## Pre-requisites
To start the app-controller backend service, pre-requisites are:

1. Linux (Preferred)
2. A kubernetes cluster installed with [knative serving components](https://platform9.com/blog/how-to-set-up-knative-serving-on-kubernetes/)
3. Link to MySQL Database (local/remote) that acts as data store for app-controller.

## Configurations
The configurations for the service are set using `config.yaml`. Sample of `config.yaml` is present at [etc/config.yaml](etc/config.yaml)

This config.yaml should be placed at `/etc/pf9/app-controller/config.yaml`, it contains: 

```
# Path to the kubeconfig file of the underlying Kubernetes cluster that has Knative installed.
1. kubeconfig path

# Database name, username, password, URL, port. 
2. DB credentials

# auth0 JWKS URL, client id.
3. auth0 credentials

# constraints on maximum apps deploy count, replical count.
4. constraints (optional)
```

## Build app-controller

Clone the repository, navigate to the cloned repository and download the dependencies using `go mod download`. Before building, ensure the `config.yaml` is configured accordingly and placed at required location.

To build the app-controller binary, use the below command, app-controller binary built using make is placed in `bin` directory.

```sh
# Using make, prefered for linux OS.
make build

# Using go build and run.
sudo go run cmd/main.go
```

### For DB Schema changes or first time builds
If DB Schema is changed or for first time builds, then to generate updated `pkgs/db/migrations_generated.go`, follow the below commands, before building the binary.

```
go get -u 'go get -u github.com/go-bindata/go-bindata/...'

# Set the path where go-bindata binary is installed.
export PATH=${PATH}:${GOPATH}/bin
cd pkg/db; go generate; cd -
```

## Run app-controller service

`app-controller` service can be run using binary and as a system service on linux machine.

### Using binary
To run app-controller through binary, follow the below command:
```sh
# Initialize and upgrade database.
./bin/app-controller migrate

# Start the app-controller service.
./bin/app-controller
```

### Using system service file (Preferred)
To run app-controller as a system service, service file [appcontroller.service](appcontroller.service) should be place at `/etc/systemd/system/` directory and app-controller binary at `/usr/bin/app-controller/` directory. To start the service follow the below commands:

```sh
# Start the app-controller service.
sudo systemctl start appcontroller.service
```

`app-controller` service will be now up and running, to check the latest status of service:

```sh
# Check the status of app-controller service.
sudo systemctl status appcontroller.service

Sample Output:
● appcontroller.service - App Controller Service
   Loaded: loaded (/etc/systemd/system/appcontroller.service; disabled; vendor preset: enabled)
   Active: active (running) since Mon 2022-02-21 10:39:19 UTC; 12s ago
 Main PID: 9144 (app-controller)
    Tasks: 16
   Memory: 9.4M
      CPU: 1.823s
   CGroup: /system.slice/appcontroller.service
           └─9144 /usr/bin/app-controller/app-controller

Feb 21 10:39:19 platform9 systemd[1]: Started App Controller Service.
``` 


* Logs for app-controller service can be found at `/var/log/pf9/app-controller/app-controller.log`

## `app-controller` APIs
To interact with the app-controller service, app-controller APIs are needed. This requires an Auth0 token.

```sh
# To get list of apps for a user.
curl --request GET --url 'http://<service endpoint>:6112/v1/apps'  --header "Authorization: Bearer ${AUTH0_IDTOKEN}" | jq .

# To describe an app by name.
curl --request GET --url 'http://<service endpoint>:6112/v1/apps/<name>'  --header "Authorization: Bearer ${AUTH0_IDTOKEN}" | jq .

# To create an app, where name is app name, image is container image of app, envs is environment variables with key:value pairs list, port is container port to access app.
curl --request POST --url 'http://<service endpoint>:6112/v1/apps'  --header "Authorization: Bearer ${AUTH0_IDTOKEN}" --data '{"name": "<appname>", "image": "<container image>", "envs": [{ "key":"<key>", "value":"<value>"}], "port": "<port>"}'

# To delete an app by name.
curl --request DELETE --url 'http://<service endpoint>:6112/v1/apps/<name>'  --header "Authorization: Bearer ${AUTH0_IDTOKEN}"

- If service is deployed locally, then can replace service endpoint with 127.0.0.1
```

## Fetching Auth0 token
Auth0 token can be fetched using Auth0 APIs. There are 3 steps to get the [auth0 id token](https://auth0.com/docs/quickstart/native/device).

1. Request device code
2. Device activation
3. Request auth0 token

### **Pre-requisites**
* [auth0 native application](https://auth0.com/docs/get-started/auth0-overview/create-applications/native-apps)
* [auth0 device code setup](https://auth0.com/docs/quickstart/native/device#prerequisites)

#### [**Request device code**](https://auth0.com/docs/quickstart/native/device#request-device-code)

```sh
# Request device code 
curl --request POST \
  --url 'https://YOUR_DOMAIN/oauth/device/code' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data 'client_id=YOUR_CLIENT_ID' \
  --data 'scope=YOUR_SCOPE' 
```
- Basic sample scope is 'scope=profile email openid'

#### **Device activation**
Upon request for device code, the [sample device code](https://auth0.com/docs/quickstart/native/device#device-code-response) response will be:
```sh
{
  "device_code": "Ag_EE...ko1p",
  "user_code": "QTZL-MCBW",
  "verification_uri": "https://accounts.acmetest.org/activate",
  "verification_uri_complete": "https://accounts.acmetest.org/activate?user_code=QTZL-MCBW",
  "expires_in": 900,
  "interval": 5
}
```

Then open `verification_url_complete` in browser, obtained from device code response to complete the device activation. 

#### [**Request auth0 token**](https://auth0.com/docs/quickstart/native/device#example-request-token-post-to-token-url)

Once device activation is successful, then request for auth0 token using below command.
```sh
# Use the device code received from device code response
curl --request POST \
  --url 'https://YOUR_DOMAIN/oauth/token' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=urn:ietf:params:oauth:grant-type:device_code \
  --data device_code=YOUR_DEVICE_CODE \
  --data 'client_id=YOUR_CLIENT_ID'
```

The response will contain both, access_token and id_token. We use auth0 `id_token` to authorize the user through app-controller. To access the app-controller APIs seamlessly export the auth0 `id_token`. 

```sh
# Replace the <id_token> with the id_token value received from request auth0 token.
export AUTH0_IDTOKEN="<id_token>"
```