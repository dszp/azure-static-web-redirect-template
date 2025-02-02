# Azure Static Web App Redirect Configuration
Source repository for a statically-configured web-redirect using an Azure Static Web App.

(See [Advanced Notes](#advanced-notes), [Troubleshooting, Tips, and References](#troubleshooting-tips-and-references), and [Glossary](#glossary) below under the Deployment steps for additional information.)

## Deployment Steps
To deploy this to a new repository/domain/Azure Static Web App:

1. Copy the contents of this repository to the new one for a new domain (to use the template repository, if defined, create a new repository and select the template repository as the source).
2. Connect a [new Azure Static Web App](https://portal.azure.com/#create/Microsoft.StaticApp) (must be logged into Microsoft Azure with appropriate permissions to use this direct Create link) to this repository with Deployment Token authentication type.
  - The Static Web App Name should be the domain name with periods replaced by hyphens to make it easy to find (periods are not allowed).
  - The Hosting Plan Type should be "Free" in almost all cases.
  - The Deployment details should have a Source of "GitHub".
  - The GitHub Organization/Repository should be the same as the one you copied the contents of this repository to, and should already exist before creating the Static Web App (Branch is always `main`).
  - The Build Presets value selected should be "Custom" which then shows these fields to fill in; change them to these if other defaults are guessed by the system:
    - The App Location value should be "`/`"
    - The "API Location" should be blank
    - The Output Location should be "`out`"
3. In the newly-created Workflow file ending in "`.yml`" in the "`.github`" folder, edit the "`jobs:`" section to add text before and after (you can optionally give a more descriptive name to the Action by editing the text after "`name:`" on the first line of the file--this is mostly useful if you follow the advanced configuration option discussed below to deploy [Multiple Static Web Apps to the Same Destination with One Repository](#multiple-static-web-apps-to-the-same-destination-with-one-repository)):

```yaml
jobs:
  build_and_deploy_job:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: true
          lfs: false
      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_KEEP_RANDOM_TEXT_HERE }}
          repo_token: ${{ secrets.GITHUB_TOKEN }} # Used for Github integrations (i.e. PR comments)
          action: "upload"
          ###### Repository/Build Configurations - These values can be configured to match your app requirements. ######
          # For more information regarding Static Web App workflow configurations, please visit: https://aka.ms/swaworkflowconfig
          app_location: "/" # App source code path
          api_location: "" # Api source code path - optional
          output_location: "out" # Built app content directory - optional
          ###### End of Repository/Build Configurations ######
```

Add this before the "`jobs:`" section with correct indentations:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Add this to the end of the "`jobs:`" section, before "`close_pull_request_job:`" with correct indentations:
```yaml
          skip_api_build: true
        env: # Add environment variables here
          IS_STATIC_EXPORT: true
```

The resulting file should now have a "`concurrency:`" section above "`jobs:`" and look similar to this from there until the "`close_pull_request_job:`" section (make sure to leave the ***(make sure you don't change the line with "`AZURE_STATIC_WEB_APPS_API_TOKEN_KEEP_RANDOM_TEXT_HERE`" in it, which will end with something different that Azure automatically created instead of "`_KEEP_RANDOM_TEXT_HERE`")!***):

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build_and_deploy_job:
    if: github.event_name == 'push' || (github.event_name == 'pull_request' && github.event.action != 'closed')
    runs-on: ubuntu-latest
    name: Build and Deploy Job
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: true
          lfs: false
      - name: Build And Deploy
        id: builddeploy
        uses: Azure/static-web-apps-deploy@v1
        with:
          azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN_KEEP_RANDOM_TEXT_HERE }}
          repo_token: ${{ secrets.GITHUB_TOKEN }} # Used for Github integrations (i.e. PR comments)
          action: "upload"
          ###### Repository/Build Configurations - These values can be configured to match your app requirements. ######
          # For more information regarding Static Web App workflow configurations, please visit: https://aka.ms/swaworkflowconfig
          app_location: "/" # App source code path
          api_location: "" # Api source code path - optional
          output_location: "out" # Built app content directory - optional
          ###### End of Repository/Build Configurations ######
          skip_api_build: true
        env: # Add environment variables here
          IS_STATIC_EXPORT: true
```

5. Edit the "`staticwebapp.config.json`" file (it must have exactly this name, and be in the root of the repository) to have these contents, replacing the text URL with the actual URL (including the `https://` prefix, like `https://redirect.to.here/path/if/you/want/one` inside the double quotes) that you would like to redirect all requests to the site to instead with a 301 permanent redirect (change the two places "301" is found to "302" to instead do a temporary redirect):

```json
{
    "routes": [
        {
            "route": "/",
            "redirect": "URL",
            "statusCode": 301
        }
    ],
    "responseOverrides": {
    "404": {
      "statusCode": 301,
      "redirect": "/"
    }
  }
}
```

6. After committing the changes to the `main` branch, the Actions tab for the repository on GitHub.com will show a new Action running, and will soon complete deployment. If there are no errors, you can visit the Static Web App URL found under the "View your application" on the Static Web App Overview page in Azure and any or no path after the site should redirect to the "`URL`" in the "`staticwebapp.config.json`" file above, which you should confirm.
7. Add Azure Static Web App Custom Domains after testing: In the Azure Static Web App config under Settings->Custom Domains, add two domains names (without the "`www`" subdomain and with the "`www`" subdomain, both) and validate them via DNS as requested. When they are validated, point the "`A`" record to the provided IP address and add a "`CNAME`" record pointing to the hostname used in the test URL above ending in "`.azurestaticapps.net`". (see the "[Custom Domain Apex `A` Record](#custom-domain-apex-a-record)" section below for more information and documentation link).
8. When both domains are validated, click the "`.azurestaticapps.net`" hostname and click one of the custom domains and click "set as default" at the top. (Once a custom domain is set as default, you must "unset" it as Default before you can switch to another as default without error.)
9. You may need a new browser, or to wait for DNS timeouts to expire, before confirming that the custom domains now redirect like the provided test URL did, completing the setup.

Because free Azure static Web Apps only allow two custom domains each, you will need to make a new one for other domains (even paid ones are limited to six custom domains, or three including with and without `www.` subdomains).

## Advanced Notes

### Multiple Static Web Apps to the Same Destination with One Repository
If you need to redirect multiple Static Web Apps to exactly the same destinations, you can use one repository with differently-named `AZURE_STATIC_WEB_APPS_API_TOKEN_KEEP_RANDOM_TEXT_HERE` tokens (in Repository Secrets) that correspond to differently-named `.yml` workflow files in the `.github` folder. Each time a change is committed to the repository's `main` branch, ***all*** of the workflows that exist will run, copying the identical files from the repository to each separate Web App. This means the destination redirects to ***will*** be the same for all of the multiple Web Apps as they are using the same source! This may be OK but should require careful consideration as additional repositories cost nothing, although they create slightly more overhead to manage.

You should be somewhat sure the redirect for all domains will go to the same location for an extended time, and name the repository with an approximation of the destination rather than the single domain name so it's use for multiple domains is clearer.

### Forwarding to Additional Paths
Another advanced option would be to adjust the "`staticwebapp.config.json`" file to redirect different paths to different destinations instead of all to the same, which would be configured using more complex syntax in the "`staticwebapp.config.json`" file.

## Troubleshooting, Tips, and References

### Custom Domain Apex `A` Record
Finding the IP address to add for the apex/root domain **(`CNAME`s cannot be used at the root of a domain!),** follow the directions in [Set up an apex domain in Azure Static Web Apps](https://learn.microsoft.com/en-us/azure/static-web-apps/apex-domain-external#set-up-with-an-a-record) including the directions for finding the "`stableInboundIP`" setting for a specific Static Web App (must be checked separately for each one). The "`www`" subdomain should be created as a `CNAME` instead of an `A` record, as it allows for better load balancing and the IP address for the apex is a necessary workaround.

### Fix Missing Deployment Token
If the GitHub Action cannot complete because the secure Deployment Token that's used as a Repository Secret that allows the Action to authenticate to the Static Web App to upload files, the steps at [Reset deployment tokens in Azure Static Web Apps]([url](https://learn.microsoft.com/en-us/azure/static-web-apps/deployment-token-management)) describe how to get a new token from the Static Web App and save it into the correct GitHub Repository Secret to re-enable the authentication. GitHub will not display the secret to you, only allow you to save a new value, to the named secret. The secret is named "`AZURE_STATIC_WEB_APPS_API_TOKEN_KEEP_RANDOM_TEXT_HERE`" where "`_KEEP_RANDOM_TEXT_HERE`" is custom for each Static Web App and matches the token name in the GitHub Workflow file.

**Note:** The Deployment Token is stored in GitHub using GitHub Repository Secrets. To find these, go to the Repository->Settings tab, on the left-hand side menu under the Security heading expand the "Secrets and variables" section and choose the Actions item. The Repository secrets will be listed there (matching the "`AZURE_STATIC_WEB_APPS_API_TOKEN_`" name prefix with a unique ending for each workflow file that corresponds with a specific Azure Static Web App) and you can click the edit button next to one to provide an updated value (but not see the current value).

### Additional Helpful Links
Generic [Azure Static Web App Build Configuration]([url](https://learn.microsoft.com/en-us/azure/static-web-apps/build-configuration?tabs=identity&pivots=github-actions)) is fully documented in how it's configured via GitHub Actions, and GitHub has separate [Deploying to Azure Static Web App]([url](https://docs.github.com/en/actions/use-cases-and-examples/deploying/deploying-to-azure-static-web-app)) documentation as well with useful Workflow configuration reference.

The static build details were taken from this [YAML GitHub Actions]([url](https://learn.microsoft.com/en-us/azure/static-web-apps/deploy-nextjs-static-export?tabs=github-actions)) example, although we are not building a Next.js application here (we could, if we wanted more control over the Static Site than just redirects).

Additional notes were pulled from the [Deploy hybrid Next.js websites on Azure Static Web Apps (Preview)]([url](https://learn.microsoft.com/en-us/azure/static-web-apps/deploy-nextjs-hybrid#enable-standalone-feature)) documentation from Microsoft.

## Glossary
- **GitHub Action:** In a GitHub repository, an Action is executed to run an automated script to do things like build and deploy an output like a website from the contents of a repository on a trigger, such as an update being committed. The Action runs on a GitHub-provided "runner" virtual machine that is created and then destroyed when finished. The latest Ubuntu Linux version is used by default. Free GitHub accounts and organizations have a couple thousand "Action Run" minutes available for free ever month; plenty to handle occasionally deploying from several repositories to live sites in Azure (since changes are infrequent and deployments usually take less than one minute each). An Action can be manually disabled if you go to the Actions tab, click the name of the Action at the left (the name is defined inside the Workflow file at the top as the "name:" and then in the top-right three-dots menu, choose Disable Workflow). When disabled, it  will not run again until you change this back to Enable Workflow.

- **GitHub Workflow File:** In a GitHub repository, files stored in a subfolder of the root `.github` folder in a repository where the filename ends in `.yml` and the file is a text file that's formatted in the YAML (Yet Another Markup Language) format in a special way to tell GitHub Actions how to run a build and deploy action. The existence of a Workflow file that's valid immediately results in a GitHub Action listed on the Actions tab that runs on the defined triggers in the file.
