# How We Work

Our team is unusually flexible, with members coming and going from client engagements, internships and exchange programs. As such, we operate under the lightest possible framework to ensure we complete our work efficiently. A light process also encourages and facilitates external contributions. This document provides a high-level summary of our lightweight processes.

## Project Tracking

* We use GitHub for all issue and project tracking 
    * Every aspect of our work, including new features, bug fixes, research topics and documentary tasks must have a GitHub issue in a Common Tools project
    * All current tasks must be 'In Progress' on our [GitHub project board](https://github.com/orgs/samsung-cnct/projects/1) 
* Meetings
    * Weekly one-hour planning meetings alternate between grooming and sprint planning
        * Grooming** - **backlog grooming to set and adjust priority levels of issues and bring desired issues into GitHub project 'Backlog' swim lanes (the first swim lane for the project boards) 
        * Sprint Planning - these weeks we prioritize backlog project tasks and order them in the ToDo swim lane
    * Daily 15-minute standups allow team members to keep others apprised of their current issues 

## Common Tools Workflow

We welcome all types of contributions from the community. We just ask you follow these steps when contributing to the project:

1. Fork the target repository into your own GitHub space
2. Create a feature branch with your work off of a current master 
3. Submit a PR from your feature branch against the master in the target repo
4. Wait for a Samsung CNCT team member to review and merge

Please refer to the kraken-lib Contributing document for more specifics on our process.  

## Projects

Common Tools is responsible for the following repositories, subject to change as we evolve.  

* [kraken-lib](https://hanjin.atlassian.net/wiki/pages/createpage.action?spaceKey=CoT&title=github.com%2Fsamsung-cnct%2Fk2&linkCreation=true&fromPageId=66584577) 
* [kraken](https://hanjin.atlassian.net/wiki/pages/createpage.action?spaceKey=CoT&title=github.com%2Fsamsung-cnct%2Fk2cli&linkCreation=true&fromPageId=66584577) 
* [kraken-tools](https://hanjin.atlassian.net/wiki/pages/createpage.action?spaceKey=CoT&title=github.com%2Fsamsung-cnct%2Fk2-tools&linkCreation=true&fromPageId=66584577)
* [kraken charts](https://hanjin.atlassian.net/wiki/pages/createpage.action?spaceKey=CoT&title=github.com%2Fsamsung-cnct%2Fk2-charts&linkCreation=true&fromPageId=66584577) 
* [Careen](https://hanjin.atlassian.net/wiki/pages/createpage.action?spaceKey=CoT&title=github.com%2Fsamsung-cnct%2Fcareen&linkCreation=true&fromPageId=66584577) 
* [kraken CI jobs](https://hanjin.atlassian.net/wiki/pages/createpage.action?spaceKey=CoT&title=github.com%2Fsamsung-cnct%2Fkraken-ci-jobs&linkCreation=true&fromPageId=66584577) 

### **Common Tools Project Board**

This is our single project board. All work we are doing must be on this board and in progress.

* [https://github.com/orgs/samsung-cnct/projects/1](https://github.com/orgs/samsung-cnct/projects/1)

### **Self-Assigning Pull Request Reviews**

Members of the Common Tools team are expected to engage with PRs opened against any of the repositories we control. As a member of Common Tools, you will be added to the team @samsung-cnct/kraken-reviewers which is set as the top-level owner in the CODEOWNERS file in all Common Tools-controlled repositories. This prompts all opened PRs to appear [here](https://github.com/pulls/review-requested) (this page is driven by the logged-in user).

For any PRs you want to review, go to that PR, add yourself in the assigned field and then remove the kraken-reviewers team from the "requested for review" field. Please do not remove any specific users, the PR author may desire their specific review.

The PR will now stop showing in the 'Review requests' link and will show up in your [Assigned](https://github.com/pulls/assigned) list of PRs.

### **Need Work and There's Nothing in Backlog or ToDo on the Project Board?**

Please run the following queries in this order and pick the most serious looking issue you can handle. Preference is, always, for P0 before P1 before P2.

* [Priority 0](https://github.com/samsung-cnct/k2/labels/priority-p0)
* [Priority 1](https://github.com/samsung-cnct/k2/labels/priority-p1)
* [Priority 2](https://github.com/samsung-cnct/k2/labels/priority-p2)

### **Feedback**

This workflow is not set in stone. If you see issues or believe something has gone awry (project list out of date, workflow is wonky or the overall organization isn't achieving the desired goals), please bring it up as soon as possible. Either in a daily standup, planning meeting or in the #team-tooltime Slack channel. Make sure you tag Pat Christopher (@patthec).

