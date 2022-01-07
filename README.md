
## Intro
Here we're going to introduce how to automate **CI/CD** *(Continous Integration/Delivery)* with **GitHub Actions** for **Salesforce** ☁️ , it will make your development, deployment and testing a bit faster and save off your time on deployment to (and between) orgs considering tracking changes is mandatory for your development process.

------------
### Expected Results
What is expected to be is when you commit changes to the feature branch and open a pull request to upper-level branch (uat, master, production, etc...), it'll be validated and then if everything is ok you'll be able to merge and considering still no errors, your changes will be deployed to org.  (check screenshot illustration)

```diff
- pink dashed lines are validation
```
```diff
+ green dashed lines are deployment
```
![alt text](https://github.com/tavaresnip/salesforce_ci-cd_automation/blob/setup/project/img/cicd2.png)

### How it works
Whenever you do an action if you wish, its possible to run a workflow, for example, on push (action) you can actually set to perform some automatically, lets see what we working on it.

------------


#### 1. Base Workflow
> For every workflow file in our project, we'll have some basics common steps and ill explain here:


------------

```yaml
## Name: actually a label that appears on actions tab when you're running an action
name: Salesforce Workflow Developer Org
## on which cases you'll run this workflows
on:
  ## "push" means on push scenarion, you can  use pull_request, push, schedule check      			out doc ¹ 
  push:
           ## this specify which branches it will be triggered, in this case pushed to branch_name  
    branches: 
      - 'branch_name'
           ## this means which folder (you can use wildcards for it) you'll be monitoring to trigger workflow  
    paths:
      - 'force-app/**'
```
##### ¹ [github actions triggers documentation](https://docs.github.com/pt/actions/learn-github-actions/events-that-trigger-workflows "github actions triggers documentation")
------------
```yaml
 ## 1. Install Salesforce CLI
    - name: 1. INSTALL SALESFORCE CLI
      run: |
        npm install sfdx-cli
        node_modules/sfdx-cli/bin/run --version
        node_modules/sfdx-cli/bin/run plugins --core

    ## 2. INSTALL DELTA SALESFORCE PLUGIN (Non-official)
    - name: 2. INSTALL DELTA SALESFORCE PLUGIN
      run: |
        echo y | node_modules/sfdx-cli/bin/run plugins:install sfdx-git-delta

    ## 3. INSTALL YQ
    - name: 3. INSTALL YQ
      run: |
        pip3 install yq
```

#### 2. Validating your build
`./github/workflows/wf_master_validate.yml`
> [Link to open file](http://aaa "Link")

 Step by Step
#### 2.1 When you need to validate a package previously deployment, we have a example like this:
------------
##### 2.1.1 - Get modified files
-------------
```yaml
    - name: 4. GETTING DIFFERENCE FROM LAST COMMIT AND BRANCH 
      run: |
 ## 1. Here we get files difference from master and uat_perf named branches
        git --no-pager diff --name-status origin/master origin/uat_perf  
 ## 2. We use delta plugin to create a package file within modified files
        node_modules/sfdx-cli/bin/run sgd:source:delta --to origin/master --from origin/uat_perf -s force-app --output .
 ## 3. Now we just print package file (debugging) 
        echo '--- Package.xml generated with added and modified metadata ---'
        echo '#FILE: package.xml :'
        cat package/package.xml
```
------------
##### 2.1.2 - Get test class to run based on your classes (used _Test standard name convention for Test Classes)
-------------
```yaml
    - name: 5. GET TEST CLASSES TO RUN
      run: |
## We use the previously 'cat' file (package.xml) and convert xml into json and save into package_json 
        xq . < package/package.xml > package_json
        echo '#FILE: package_json'
        cat package_json    
## when package_json on hands, we use jq to get all ApexClass inside and include _Test and save to TEST_CLASSES file        
        xq . < package/package.xml | jq -r '.Package.types | if type=="array" then .[] else . end | select(.name=="ApexClass") | .members | if type=="array" then join("_Test,"), join(",") else . + "," + . + "_Test" end' > TEST_CLASSES
## print file to check if its correct
        echo '--- TEST CLASS TO RUN ---'
        echo '#FILE: TEST_CLASSES BF'
        cat TEST_CLASSES
        echo '#FILE: TEST_CLASSES AF'
## remove line break from TEST_CLASSES file for merging lines and save into TEST_CLASSES_MERGED
        tr '\n' , < TEST_CLASSES > TEST_CLASSES_MERGED
        cat TEST_CLASSES_MERGED
```
------------
##### 2.1.3 - Authorize your github with your org credentials
###### Reference used: [LINK #1](https://atrium.ai/resources/how-to-implement-salesforce-ci-cd-with-github-actions/) --- [LINK #2](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_auth_sfdxurl.htm#cli_reference_auth_sfdxurl_store)
-------------
Use your VSCode to get your info, on terminal use: sfdx force:org:display –targetusername –verbose
and you'll get some info on output and we'll get only "Sfdx Auth Url", copy value into a SECRET on your repository config
. Go to Repo Settings -> Secrets -> New repository Secret -> copy value into "Value" and name as you want, my example as "SFDX_LR_TEST_URL" and we'll use it later. Now on yaml file lets break it down:
```yaml
## 6. SETUP CREDENTIALS (SETUP YOUR URL ON SECRET) BASED ON SFDX_LR_TEST_URL PREVIOUSLY SETUP AND SAVE ON SFDX_QA FILE  
    - name: 6. SETUP CREDENTIALS 
      shell: bash
      run: 'echo ${{ secrets.SFDX_LR_TEST_URL}} > SFDX_QA'
```
##### 2.1.4 Authenticate on dev hub (check if its enabled on your org)
```yaml
 ## 7. AUTHENTICATE ON DEV HUB USING SFDX_QA 
    - name: 7. AUTHENTICATE ON DEV HUB
      run: node_modules/sfdx-cli/bin/run force:auth:sfdxurl:store -f SFDX_QA -s -a LRQA
```
##### 2.1.5 Now we're going to deploy our changes:
*** 
###### first you need to understand how it work on two differents scenario: new and modified files or removed files, on deployment it works normally, delivering all files and running test classes but on removing files it will occur on a further separated process that will get "destructiveChanges" folder to identify removed files, for validate and deploy we only set "--checkonly" on commando to validate, lets put hands on:
##### 2.1.5.1 - Validate for new/modified files and report 
```yaml
## HERE WE GET TEST_CLASSES_MERGEDFILES AND RUN force:source:deploy related to package/package.xml
## we run -l RunSpecifiedTests to run only test that are inside file
## set --loglevel ERROR but it optional
## set --checkonly it means that nothing will be deployed, process will works as "Validate" mode 
    - name: 8. VALIDATE BUILD ON SALESFORCE => Added & Modified files
    run: |
      echo '############ VALIDATE BUILD ON BEFORE DEPLOYMENT ############'
      cat TEST_CLASSES_MERGED
      node_modules/sfdx-cli/bin/run force:source:deploy -x package/package.xml -l RunSpecifiedTests -r "$(<TEST_CLASSES_MERGED)" --loglevel ERROR --checkonly
```
##### 2.1.5.2 - Get report about how validate process finished
```yaml
## Here I have a advice, we use normal force:source:deploy:report within --verbose setup
## but i've included 'if: always()' because if your validation failed next steps will be skipped and we need to know why it has failed and it'll prevent to break and show your fail reasons
   - name: 9. REPORT FROM DEPLOY (ADDED & MODIFIED FILES)
     if: always()
     run: |
       node_modules/sfdx-cli/bin/run force:source:deploy:report --verbose
```
##### 2.1.6 - Validate files and get report
##### 2.1.6.1 - Validate removed files
```yaml
## Here we use same strategy but with some changes:
## 1. we use a different xml created by delta command on previous steps (commando create package and destructiveChanges)
## 2. and we use mdapi to validate
## 3. set --checkonly for only validating
   - name: 10. VALIDATE BUILD ON SALESFORCE => Removed files
     run: |
     echo '############ VALIDATE BUILD ON BEFORE DEPLOYMENT (REMOVED FILES) ############'
     cat destructiveChanges/destructiveChanges.xml
     node_modules/sfdx-cli/bin/run force:mdapi:deploy -d destructiveChanges --ignorewarnings --    loglevel INFO --checkonly
```
##### 2.1.6.2
```yaml
## Use same report commando to get results
   - name: 11. REPORT FROM DEPLOY (REMOVED FILES)
     if: always()
     run: |
     node_modules/sfdx-cli/bin/run force:source:deploy:report --verbose
```
***
Now you can check at beginning on #2 section contains wf_master_validate.yml file and this file is ready to trigger a workflow whenever a pull_request start to master branch on 'force-app/' folder, this workflow will get modified files (new/modified/removed) from uat_perf -> master and will validate all meta data. 
Now for next steps we just use same workflow base and can deploy files removing '--checkonly', your deployment will works after you pushed a pull_request, sou you need to change it to 'push' and change your target path to get modified files as snippet codes shows below, we can set flow as: 
 - Get Modified Files => Validate => Deploy => Report => Run Test Classes 
Additional Snippest:
##### 1.  Run Test Classes - [SFDX Documentation](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_testing.htm)
```yaml
    ## 14. CHECK FOR APEX TEST -- TEST BEFORE DEPLOY TO CHECK IF EVERYTHING IS OK (DISABLED)
    - name: 14. RUN APEX TESTS
    run:
    node_modules/sfdx-cli/bin/run force:apex:test:run --resultformat tap --codecoverage -c -r human
```
##### 2. Get difference from last commit to previous one
```yaml
## 4. GET MODIFFIED FILES BETWEEN LAST 2 COMMITS

   - name: 4. GETTING DIFFERENCE FROM LAST COMMIT AND BRANCH
   run: |
    node_modules/sfdx-cli/bin/run sgd:source:delta --to 'HEAD' --from 'HEAD^' -s force-app --output .
    echo '--- Package.xml generated with added and modified metadata ---'
    echo '#FILE: package.xml :'
    cat package/package.xml
```
----
Workflows Templates:
```diff
+ .github/workflows/wf_master_validate.yml
  ```
|Triggered by Event: |PULL_REQUEST when 'opened'|
|--|--|  
|Branch Name : |uat_perf
Path : |force-app
Actions : | Validate Package & Get Results
Target : | difference from 'develop1' branch from 'uat_perf'

```diff
+ .github/workflows/wt_master_deploy.yml
```
  
|Triggered by Event: | PUSH |
|--|--|  
|Branch Name : |uat_perf
Path : |force-app
Actions : | Validate, Deploy Package & Get Results
Targets : | difference between last commit and last but one commit on uat_perf

```diff
- some ref. links related:
```
* [Delta Repository](https://github.com/scolladon/sfdx-git-delta)
* [Another repository with templates](https://github.com/olopsman/youtube-sfdx-delta-deploy/tree/ADO-111)
* [Tutorial without Delta](https://atrium.ai/resources/how-to-implement-salesforce-ci-cd-with-github-actions/)
* [Repository as Example without Delta](https://gist.github.com/garvitkhandelwal-atrium/699cbb624e5d1805b9de14e99b0dc964)
