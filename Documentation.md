This is documentation about the Nginx configurations to do static resources separation and reverse proxy from one instance to another (based on Vue frontend project)
## User issue
### Goal
Set up nginx reverse proxy for the deployed graph and 10k pages, to make them subpages of the overall landing page.

### User Story
As a client, in the Landing Page, I want to forward to different project pages that were deployed on different instances, with the same URL prefix as the Landing Page. 

### Definition of Done
For example, if the landing page is on http://landingpage:80, project1 and project2 should be proxied from another instance, and act as http://landingpage/project1 and http://landingpage/project2.

### Problem Statement
1. The common static resources could be deployed separately on the proxy server for efficiency, and then be accessed by the public with a URL so that we can fetch them in any further projects.
2. For the independent resources, we want each project to fetch its own resources from a certain path, it's important to configure the path before doing Nginx Proxy.

## Solution
### Separate Static Resources
