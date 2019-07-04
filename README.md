# IAM

This is the source repository for all of Cvent's CloudFormation IAM templates and the entry point for automation for accounts supported by ICR. Instructions on how to create and update IAM CloudFormation templates can be found 
[here](https://wiki.cvent.com/pages/viewpage.action?pageId=133504905). 


# Workflow
A user is expected to have an approved ICR Jira ticket ([ICR Project](https://jira.cvent.com/projects/ICR)). The user is then able to submit a PR with a branch using the same name as the Jira ticket number. For example an ICR ticket with the number **ICR-XXXX** would have a corresponding branch named **ICR-XXXX**.

Once the PR has been submitted a build will kick off with basic linting and rule checking. You can check whether it passed by looking at the build status in the top right of the corner in the PR.

![picture](images/build.jpg)


If the build fails you can click on the build link (highlighted in red above) to drill down into the Jenkins job to see why it failed.

Once the build has passed and the PR has been approved, it can be merged into master. Upon merging into master the aws-update-cloudformation-stack will be kicked off. You can check the status of the job through the [Cloudops Jenkins](https://cloudops-jenkins.core.cvent.org/) server. If the job failed please check the aws-update-cloudformation-stack console output and the AWS CloudFormation stack(s) to see why and submit another PR with the fix.

## Workflow Summary
![picture](images/sg_automation.jpg)

## Owners

### Cloud Automation

Their wiki space can be found [here](https://wiki.cvent.com/pages/viewpage.action?pageId=50961668)
