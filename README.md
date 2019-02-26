# AWS Track & Trace Sample

Sample system to track and trace vehicles and assets in the cloud. This project contains infrastructure definition and sample code for spinning up a web user interface with a map, in which you can connect your existing or simulated vehicles and assets, and visualize how they progress through their journey.

* [Getting started](#getting-started)
  * [Prerequisites](#prerequisites)
  * [Deploying the infrastructure](#deploying-the-infrastructure)
    * [Configuring the project](#configuring-the-project)
    * [Preparing the modules](#preparing-the-modules)
    * [Deploying the DNS Information](#deploying-the-dns-information)
    * [Main infrastructure](#main-infrastructure)
    * [Post deployment tasks](#post-deployment-tasks)
  * [Delivering the solution](#delivering-the-solution)
    * [Get a Google Maps API Key](#get-a-google-maps-api-key)
    * [Preparing and deploying the UI](#preparing-and-deploying-the-ui)
* [Using the system](#using-the-system)
  * [Fleet map](#fleet-map)
  * [Onboarding assets](#onboarding-assets)
  * [Customizing assets](#customizing-assets)
    * [Configuring asset style](#configuring-asset-style)
    * [Using asset sensors](#using-asset-sensors)
* [License Summary](#license-summary)
  * [Graphical content](#graphical-content)

## Getting started

This project uses the [AWS Cloud Development Kit](https://github.com/awslabs/aws-cdk) - aka CDK - for describing the infrastructure resources needed. 

### Prerequisites

Before starting deploying this solution, you will need to have the following resources:

* **AWS Account:** If you don't have one, go to [https://aws.amazon.com/]() and create one.

Additionally, you will need the following software installed in your workstation:

* **AWS CLI:** You will need to install the [AWS CLI](https://aws.amazon.com/cli/) and configure your access keys to interact with your AWS account. _NOTE: You will need Python and pip installed to do this._
* **AWS CDK:** Follow the instructions [here](https://github.com/awslabs/aws-cdk) to install the Cloud Development Kit. _NOTE: You will need to have `Node.js` and `npm` installed to do this._
* **jq:** You need the [`jq`](https://stedolan.github.io/jq/) cli to configure your system.
* **Clone this repository:** Get the contents of the project in your machine.

### Deploying the infrastructure

The infrastructure for this solution must be deployed complying with the following flow:

![Deployment flow](/static/deployment-flow.png)

#### Configuring the project

Once you have a local copy of the project in your local machine, you need to configure the required parameters for deploying the infrastructure. Navigate to the `infra/` folder and open the `config.sample.ts` file.

```ts
let Config: ConfigModel = {
  general: {
    solutionName: 'AWSTrackAndTrace',
    description: 'Sample project for tracking your moving assets.'
  },
  administrator: {
    username: 'admin',
    email: 'john.doe@example.com',
    name: 'John Doe'
  },
  dns: {
    domainName: 'myfleet.example.com',
    hostedZoneId: 'AABBCCDD'
  }
}
```

* **General:** Basic information about the deployment.
* * **Name:** Human friendly name of your solution.
* * **Description:** Description of your solution.
* **Administrator:** Information about the administrator user.
* * **Name:** Full name of the administrator user.
* * **Email:** Email address of the administrator user.
* * **Username:** User name for the admin user.
* * **Phone:** Phone number of the admin user.
* **Dns:** _[OPTIONAL]_ Information about the DNS records.
* * **Fqdn:** Domain name of the solution. If you specify a Dns configuration, you **must** provide this value.
* * **Hosted_zone:** _[OPTIONAL]_ ID of the existing hosted zone.

Configure the values to your best convenience and store the result at `infra/config.ts`. This file will configure your deployment once we move forward.

#### Preparing the modules

The infrastructure of this project is completely modular. All infrastructure is stored under the `infra/` folder, and modules can be found at the `packages` folder inside it. The `core` folder puts all modules together into a CDK Application, that will be responsible for deploying the infrastructure as a whole.

We have created a script that will help you autommatically build all modules and infrastructure definition, so its ready to be deployed without many extra steps. From the project folder on a terminal, simply run `npm run build:infra`.

#### Deploying the DNS Information

_NOTE: This process is only neccessary if you want the system to be accessible from your own domain - e.g. `myfleet.example.com`. If you'd rather have an autogenerated URL for your system - e.g. `xxyyzz.cloudfront.net` - you can move to the next stage._

The first deployment we will deploy is our DNS information, which consists on an ACM Certificate and a Route53 Hosted Zone - if you don't already have one. We deploy these resources separately as the certificate needs to be in the North Virginia region to be successfully used by CloudFront.

_NOTE: As part of the certificate creation the system will attempt to send an email to several addresses that you need to receive in order to confirm the creation. To understand more about this process, please follow [this guide](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate-email.html). The system won't finish deploying until the certificate is verified and issued._

* Navigate to the `infra/dns` folder.
* Run `cdk deploy`, and confirm the operation.

Once the CDK finishes deploying, the tool will output some values we will need to configure. You will see something like this:

```bash
✅  AWSTrackAndTrace-DNS

Outputs:
AWSTrackAndTrace-DNS.HostedZoneId = AABBCCDDEE
AWSTrackAndTrace-DNS.CertificateArn = arn:aws:acm:us-east-1:123456789012:certificate/aaaaaaa-bbbbbbb-cccccccc
```

* `HostedZoneId`: Id of the Hosted Zone. If you already have a defined hosted zone in your `infra/config.ts` file, you won't need this value - as it will be the same you defined.
* `CertificateArn`: The Arn of your recently created certificate. You will need to copy this value into your `config.ts` file:

```ts
let Config: ConfigModel = {
  ...
  dns: {
    certificateArn: 'arn:aws:...',
    hostedZoneId: 'AABBCC...',
    ...
  }
}
```

#### Main infrastructure

This process will deploy the infrastructure responsible of serving your system.

* Navigate to the `infra/core` folder.
* Run `cdk deploy`, and confirm the operation.

_NOTE: Optionally, you can run `cdk synth` to output a CloudFormation template of your infrastructure definition, to spin it up using CloudFormation directly._

This will kickstart the deployment process, which may take about 15-20 minutes to complete. Time for a coffee :). Then, you need to open the file `scripts/config.sample.sh` and set the following values - most of them present at `infra/config.ts`. Then save the file as `scripts/config.sh`. These values will be used from the delivery scripts to configure the system prior delivery.

```sh
export STACK_NAME="AWSTrackAndTrace" # Map this from infra/config.ts > general.solutionName
export AWS_REGION="eu-west-1" # Change this if you deploy in a region different than Ireland
```

The last configuration process before being ready to deliver is to store the infrastructure resource identifiers in a file so our delivery scripts can fetch them. From the project's root folder, run `npm run configure:infra`.

#### Post deployment tasks

After deploying the project, we need to execute certain tasks to fully configure our system. These tasks are unfortunately not creatable within the CDK, so we have prepared a script that runs these tasks for you:

* **Request a Cognito Login domain:** You will select a unique domain name that will be used for login - your login domain will be of the form `<your-domain-name>.auth.<region>.amazoncognito.com`. You can read more information about Cognito custom domains [here](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-ux.html).
* **Configure app client settings:** Prepare your Cognito User Pool to authenticate your users through the client. [More information](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-app-settings.html).
* **Configure token-based role mappings:** Your Identity Pool supports giving dynamically roles to your users based on your token claims. 

From the project's root folder, run `npm configure:post-deployment <your-domain-name>`.

Once you have executed this task, open the file `scripts/config.sh` and reflect your domain name there:

```sh
...
export COGNITO_CUSTOM_DOMAIN="<your-domain-name>"
```

### Delivering the solution

Once your infrastructure is ready, we need to configure and deliver the UI code into it so we could access it and use it.

#### Get a Google Maps API Key

Follow the instructions at the [Google Developers](https://developers.google.com/maps/documentation/javascript/get-api-key) page to generate an API Key for Google Maps (for Javascript). Once you have it, configure your `scripts/config.sh`:

```sh
...
export GOOGLE_MAPS_API_KEY="<your-maps-api-key>"
```

#### Preparing and deploying the UI

In order for the UI to fully comprehend the underlying infrastructure, we need to configure it reflecting the required resource identifiers. All these identifiers have been defined as outputs in the infrastructure definition, so we could automate the configuration process. Navigate to the project's root folder and run `npm run configure:ui`.

Once your UI finishes configuring, you are ready to deploy it. From the project's root folder, run `npm run deploy:ui`.

Is the UI not deploying correctly? Take a look at the [TROUBLESHOOTING](./TROUBLESHOOTING.md#failed-during-ui-deployment) section.

When your deployment finishes successfully you could start testing it by accessing its URL. If you have configured your custom domain you could simply navigate to it - e.g. `myfleet.example.com`. Otherwise you'll need to retrieve the distribution URL, which will be the entry point to your solution. From the project's root folder execute `npm run get:entry`. It will print the distribution URL to the standard output.

## Using the system

Type the URL you've gathered previously on a browser. You will be redirected to your recently deployed system, more specifically to the UI's landing page. 

![Landing page](/static/landing-page.png)

Click on the _Access_ button to start the authentication process. You should be redirected to the Cognito login page. In there you can use the username you've defined in the `infra/config.ts` file, along with a password that you should've received over email. Use those credentials and log in. You should be asked to change your password, and use your new credentials to finalize the login process. Once you've done that, you should be redirected back to the private section of the solution's UI.

![Dashboard Overview](/static/dashboard-overview.png)

### Fleet map

The UI is a Single Page Application that renders a fullscreen map and certain tools to interact with the solution. The first time you access the map it should have no assets present. You should be able to zoom and pan the map to your desire.

TODO More here

### Onboarding assets

You can add as many assets as you want to be shown in your map. An asset is - in essence - an [AWS IoT Thing](https://docs.aws.amazon.com/iot/latest/developerguide/iot-thing-management.html). This _Thing_ will have a state object that will handle the status of your asset at all times. 

The system has a tool for adding these _Things_, that you can find on the config tab - the _cog_ icon on the left overlay menu. This tool will help you reference your Things, and map the `location` variable from those Things' states. This variable will help the system render your Thing in the map, and subscribe to its state-change events so the map item updates automatically when there's a location change.

![Add asset menu](/static/add-asset-menu.png)

Input the **Thing name** in the first field, and the `location` variable in the second. The location variable should be the path to where the actual asset latitude and longitude are stored, starting from the reported state object - stored in the device's shadow. The location object should have a similar appearance than this: 

```json
{
  "latitude": 1.23,
  "longitude": 4.56
}
```

Once you have these values set up, you can click on the _Onboard_ button. You should see a dark dot appearing in the map immediately, and a notification once the system has persisted the asset information in the backend storage.

**Sample device onboarding**

![Sample device onboarding](/static/add-asset-sample.png)

![Sample device attributes](/static/add-asset-sample-attrs.png)

* **Thing name:** `AWSTrackAndTrace_TestDevice1`
* **Location variable:** `systems.gps.location`

![Sample device onboarded](/static/add-asset-sample-result.png)

### Customizing assets

Once you have onboarded several assets, you would want to give them additional styling configuration so you could identify them easier. To start customizing your assets, simply click on their marker on the map.

![Customize assset overview](/static/customize-asset-overview.png)

#### Configuring asset style

You can individually assign a style to each asset, that will be used by default to render such asset's marker. The marker style must have the following structure:

```json
{
  "path": "CIRCLE",
  "scale": 8,
  "fillColor": "#444",
  "fillOpacity": 1,
  "strokeWeight": 0,
  "strokeColor": "#444"
}
```

#### Using asset sensors

_Sensors_ are data definitions on your device's state, normally produced by real-world's sensors - e.g. temperature and humidity, accelerometer forces, gps locations, etcetera. We will map these state values to defined sensor names and data types.

To add a sensor, select a **unique name** for the sensor, a _Sensor type_, the location of the value field - starting from the `state` variable - and the Units, and click _Save_.

**Example:**

* **Sensor name:** CoolerTemperature
* **Sensor Type:** Number
* **Value Field:** `systems.cooler.temperature`
* **Value Units:** `ºC`.

Once the sensor is saved, we could use it to create style conditions. Style conditions can modify the appearance of your assets if the condition expression is matched - e.g. set the `fillColor` to `blue` if temperature goes below 15ºC. Conditions are evaluated in order, and only style overrides need to be defined - i.e. you don't need to define the full style object, but only the properties you'd want to override from the default style.

**Example:**

* **Condition Expression:** `CoolerTemperature < 15`
* **Style Override:** `{ "fillColor": "blue" }`.

![Map with assets](/static/map-with-assets.png)

## License Summary

This sample code is made available under a modified MIT license. See the [LICENSE](./LICENSE.md) file.

### Graphical content

* **Landing page background image:** Licensed under the [Pixabay license](https://pixabay.com/en/service/license/).
* **Logo**: Truck licensed under the [Pixabay license](https://pixabay.com/en/service/license/), modified to include the AWS logo and the solution title.
