[Google Apps (G Suite)](https://developers.google.com/identity/protocols/OpenIDConnect), [Microsoft Azure AD](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-protocols-oauth-code), and [GitHub](https://developer.github.com/apps/building-oauth-apps/authorization-options-for-oauth-apps/) authentication for [CloudFront](https://aws.amazon.com/cloudfront/) using [Lambda@Edge](http://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html). The primary use case for `cloudfront-auth` is to serve private S3 content over HTTPS without running a proxy server to authenticate requests.

## Description
Upon successful authentication, a cookie (named `TOKEN`) with the value of a signed JWT is set and the user redirected back to the originally requested path. Upon each request, Lambda@Edge checks the JWT for validity (signature, expiration date, audience and matching hosted domain) and will redirect the user to configured provider's login when their session has timed out.

## Usage
If your CloudFront distribution is pointed at a S3 bucket, [configure origin access identity](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html#private-content-creating-oai-console) so S3 objects can be stored with private permissions.

Enable SSL/HTTPS on your CloudFront distribution; AWS Certificate Manager can be used to provision a no-cost certificate.

Session duration is defined as the number of hours that the JWT is valid for. After session expiration, cloudfront-auth will redirect the user to the configured provider to re-authenticate. RSA keys are used to sign and validate the JWT. If the files `id_rsa` and `id_rsa.pub` do not exist they will be automatically generated by the build. To disable all issued JWTs upload a new ZIP using the Lambda Console after deleting the `id_rsa` and `id_rsa.pub` files (a new key will be automatically generated).

#### Login using Github

1. Clone or download this repo
1. Navigate to your organization's [profile page](https://github.com/settings/profile), then choose OAuth Apps under Developer settings.
    1. Select **New OAuth App**
    1. For **Authorization callback URL** enter your Cloudfront hostname with your preferred path value for the authorization callback. Example: `https://my-cloudfront-site.example.com/_callback`
1. Execute `npm run-script build` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
    1. Choose `Github` as the authorization method and enter the values for Client ID, Client Secret, Redirect URI, Session Duration and Organization
        -  cloudfront-auth will check that users are a member of the entered Organization.
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

#### Login using Google

1. Clone or download this repo
1. Go to the **Credentials** tab of your [Google developers console](https://console.developers.google.com)
    1. Create a new Project
    1. Create an **OAuth Client ID** from the **Create credentials** menu
    1. Select **Web application** for the Application type
    1. Under **Authorized redirect URIs**, enter your Cloudfront hostname with your preferred path value for the authorization callback. Example: `https://my-cloudfront-site.example.com/_callback`
1. Execute `npm run-script build` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
1. Choose `Google` as the authorization method and enter the values for Client ID, Client Secret, Redirect URI, Hosted Domain and Session Duration
1. Select the preferred authentication method
    1. Hosted Domain (verify email's domain matches that of the given hosted domain) 
    1. JSON Email Lookup
        1. Enter your JSON Email Lookup URL (example below) that consists of a single JSON array of emails to search through
    1. Google Groups Lookup
        1. [Use Google Groups to authorize users](https://github.com/Widen/cloudfront-auth/wiki/Google-Groups-Setup)
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

#### Login using Microsoft Azure

1. Clone or download this repo
1. In your Azure portal, go to Azure Active Directory and select **App registrations**
    1. Create a new application registration with an application type of **Web app / api**
    1. Once created, go to your application `Settings -> Keys` and make a new key with your desired duration. Click save and copy the value. This will be your `client_secret`
    1. Above where you selected `Keys`, go to `Reply URLs` and enter your Cloudfront hostname with your preferred path value for the authorization callback. Example: https://my-cloudfront-site.example.com/_callback
1. Execute `npm run-script build` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
1. Choose `Microsoft` as the authorization method and enter the values for [Tenant](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-howto-tenant), Client ID (**Application ID**), Client Secret (**previously created key**), Redirect URI and Session Duration 
1. Select the preferred authentication method
    1. Azure AD Membership (default)
    1. JSON Username Lookup
        1. Enter your JSON Username Lookup URL (example below) that consists of a single JSON array of usernames to search through
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

### Configure Lambda and CloudFront

1. Upload **cloudfront-auth.zip** to Lambda using the [AWS Console](https://console.aws.amazon.com/lambda/home):
    1. Choose **Create function**
    1. Choose **Blueprints**
        - Search and select **cloudfront-http-redirect**
        - Enter Name (e.g. `My_Site_Cloudfront_Auth`)
        - Select **Create new role from Templates** as the **Role**
        - Enter Role Name (e.g. `Lambda_At_Edge_Cloudfront`)
        - Add the role policy template **Basic Edge Lambda Permissions**
        - Click **Create function**
    1. Choose **Upload a .ZIP file** under the **Function code** header and **Code entry type** field
        - Click **Upload** and choose your generated `cloudfront-auth.zip` file
        - Set **Timeout** under **Basic Settings** to be 5 seconds
        - Click **Save**
    1. Choose **Publish new version** under the **Actions** button
        - Enter a version description (e.g. `v1`)
    1. Copy the ARN value (required to be your published versioned) from the top-right: `arn:aws:lambda:us-east-1:9999999999:function:my-site-cloudfront-auth:1`

1. Configure CloudFront to use the Lambda function upon **viewer request**:
   1. Choose the **Behaviors** tab for your CloudFront distribution, mark the checkbox for the origin, choose **Edit**
      ![alt text](https://lh3.googleusercontent.com/T4b26lGh3yu4SSxXAG3Vb63iuWxTXkqgFTiXNp5i-NCGQ6AgH_Lal5CYse6gZJOpjSK8xKi9kuF8niPKbqjbrTFYDB7n6ZNv-mANWytL_zatFwDamFQZ_1RnDnEAGkXfrKONRNfJh6w8qjLHKuCk1JWnqsIWYnIr44J2j6wFKceasggPxnh8IfhC869-Pz3GRC6AvURWLOVoQWZI5tp7NQ6U4NGZ-dI-bEjOSTqx96PEnlbIY4r-Js76SgbKI_94aow5eMXmhbGFcsheUIZ5jRXJ6NT9Z3SpPEw0tvJwqDEs5UyM8xva_Ghb33EsV3bfDzZbaKoCXk3diKnBCV5BTpfx8szaiOxiqHZY8wfFEZfkeZi-sZECSAECcnXcIWVEGId52vjtQmNi0krfwcAUSHzkEMB3E3jHMH2fd8q3Pp8YO5w1A2wgAE_SDVuT6JRS-i1vFoRx-OkfSpNI4kdY7Uh4MxvP6fR_hNVPCxilM9y0D_S8ln7MWAPE_7V3RkV214SObk_PoU4dW3u67PD1BUfD8kR96Kf6UV8s5IhM61ks9u1PvbFj822y51CWAhTRe02tcwPdB9Km0jbYXYgzkPFkzPXCYCKeTLCg0m2m4HAUS5SL7P3ftYN98FyOdYYrbtmYiJtwatH6gjwfyX6ENc2rDMa4A8Q=w1684-h586-no "Edit behavior")
   1. Under **Lambda Function Associations**
        - Select **Viewer Request** for the **Event Type**
        - Enter the Lambda ARN value (e.g. `arn:aws:lambda:us-east-1:9999999999:function:my-site-cloudfront-auth:1`)
        ![alt text](https://lh3.googleusercontent.com/9YGTDMxX-9q_3GhW-w9ORcWejG3ZoQUBhviVb3_Dr1iCuvbmvSHM0WXLZ5UrlvUzkuDcfBtJJMqF5C7kWdJuG5P2abOiBNhLoxTF41oQqOzyWofio6TCTW_56SjjaMCzDyocusbx9GzOaJNHAWIIvDXByLwfHCaWQf7VcGdBx4WnwKwvq5_08Pv2G2JIkznTRzSrpd6KbMpkSUT7H3dOO-mZbPEl6NKvmIJ0iAW834R4KSx0gHEtzTLYu6FPN0oWHkQwGHh2x4kmBaSp1WyxaE98okVe3QMZ_bYPt2NDVSQHuPcd3mOQAjJBNnyBoq5zgJYe5r5AdSbyIJ7bfJDthUcqk_ZL67DJ39_NkFrdyJN2A5n5Iunn2axtN7vMlsi54WxfcQFpxTs3x_2QPRYGEaYUnjuLVpS7ZdlDgp3-46pUqEISCOAVb5wMU2lY4KFEdEiSOccKcvjuyK25GxvDvGkZTR5xP6DRm8A6uOmQbOEEL5M9OMB0_OS5pMW_DWAnXeqwHSLZk42Wc58YyJlLSZ0WBnFPvAHoEuV2N-mYL6NhKSoLBEK_HM6TyEH03SolS6baVyTH_cPSDwya-N7EQtnyM1aL3WKaKv6V_ETTH3g8zOB-EydUbjpEEPyUJrjqFsrHNQieeksEGIWe0gqX93r7FpxiLXk=w1528-h298-no "Point to Lambda function on viewer request")
        - **Save**
   1. It may take Cloudfront up to 20 minutes to fully configure the distribution. You may receive strange errors while the distribution is still updating.
   1. When fully deployed, your browser will be redirected to your authentication provider as necessary! Enjoy!

## Authorization Method Examples

- [Use Google Groups to authorize users](https://github.com/Widen/cloudfront-auth/wiki/Google-Groups-Setup)

- JSON array of email addresses

    ```
    [ "foo@gmail.com", "bar@gmail.com" ]
    ```

## Build Requirements
 - [npm](https://www.npmjs.com/) ^5.6.0
 - [node](https://nodejs.org/en/) ^7.10.0
 - [openssl](https://www.openssl.org)

## Contributing
All contributions are welcome. Please create an issue in order open up communication with the community.

When implementing a new flow or using an already implemented flow, be sure to follow the same style used in `build.js`. The config.json file should have an object for each request made. For example, `openid.index.js` converts config.AUTH_REQUEST and config.TOKEN_REQUEST to querystrings for simplified requests (after adding dynamic variables such as state or nonce). For implementations that are not generic (most), endpoints are hardcoded in to the config (or discovery documents).
