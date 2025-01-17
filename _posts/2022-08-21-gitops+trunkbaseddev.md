# gitops+Trunk-dev
2 hot words in software these days are [**gitops**](https://www.weave.works/technologies/gitops/) and [**trunk based development**](https://trunkbaseddevelopment.com/). I realized some gaps in adapting both at the same time as per prescriptions.

This a quick note describes gitops and trunkbaseddev and their quick pro/cons. But plenty of articles out there to know more about these.
More importantly, this note describes how I see the issues in adapting both practices at same time and possible solutions.

## What is Trunk Based Development:
It's simple as it says. Develop on trunk and deploy from trunk. For the gitflow fanatics and branching addicts, meetings freeze at the utterance of developing on one single branch. But, yes, for those who have lived , branching and merging hell, this concept doesn't sound as bad (in fact , a sigh of relief).
#### Trunk Based Branching doesn't mean - Don't branch at all:

 - But follow principle of "***branch as late as possible, merge as soon as possible, god forbid, if your branch and trunk is moving at a fast speed, reverse merge as many times into the branch***". 

When someone needs to branch, these questions deserve answers:
   1. why would you break software, which requires branch?
   2. Are there any techniques ,which would allow committing on the same branch?
   3. What can one do to ship software with unfinished code?

 This is not note on trunk based development, but couple of principles below enables trunk based development:
- Immutable components
- Interface Versioning and maintaining compatibility
- Use of feature flags

### Pro of Trunk Based Dev
- **No or minimal integration efforts** - By using these techniques, one usually follows true spirit of Continuous Integration. One has less need or less efforts to "integrate" or merge each other's code. (For those , to whom merge only means source code merge, it's not so. Integration and/or merge of 2 code bases should require test cases written for each code bases to pass, at the least unit, at best integration or end to end).
- **Ship code faster to prod** - No matter how much tested in house, real test of code is in front of end user. Not only functional test, but non-functional (perf/security) and acceptance.
- **Blue/Green, Canary deployments and rollbacks** - using feature flags, behavior of code (finished or unfinished) can be put to test very easily.

### Con of Trunk Based Dev
Trunk Based development does have a **con** as well. There is no escape from advancing parts of software at their own pace. Some do it in branches, some do it within code. By doing in code, one gets above benefits. But without proper structure and frameworks, code can get a big switch statement.

### Summary
Even with Trunk based dev and use of git, it is expected to have PR branches and hot fix branches. PR branches should not exists beyond 4 hours. Hot fix release branches should not exist beyond end of the current sprint.

### Contradicting point with gitops
Had to take a bit of dive into Trunk based Dev before I could make my pitch on combing gitops and trunk based dev practices. Trunk based dev, does not or should not, produce branches beyond PR & Release branches. Definitely not environment based branches as gitops perhaps suggests, when it prescribes state (a.k.a configuration?) to be in git (per environment?) below ( ***3. The canonical desired system state versioned in Git***)

## What's gitops:

Liking the clear and crisp definition here, [here](https://www.weave.works/technologies/gitops/) are the 4 principles:
### 1.The entire system described declaratively 
This is the one I like the most. No matter if plane strikes into towers or an asteroid destroys data centers, this principle allows (*theoretically*) to re-establish the system (software and codified infrastructure) by checkout system state from git and deploy it again. Granted there are lot of subtle assumption above, in the expectation above. e.g.
 - All data centers are "almost" equal and
 - not all data centers are destroyed and
 - the undestroyed data centers are reachable etc

There are certain principles also need to be followed when describing system state:
 - state of same system when deployed again could be different - e.g. IP address ranges could change
 - System is not data center state full

### 2. Software agents to ensure correctness and alert on divergence
There are plenty of ways of ensuring the environment drift and alert based on state drift. Perhaps it needs a note of it's own.
### 3. The canonical desired system state versioned in Git
**Issue #1 -** storing state of system in git. 

While a good principle, I think it prescribes , git for all types of states. In my experience, statefull-ness of system lies in couple places e.g.
 1. A state based infra structure provisioning system e.g. Terraform - hence tf state files
 2. Secure and un secure configuration
Thankfully, the most core STATE of the system, Business Data , is not assumed to be in git. It's a function of a database (any kind relational or unstructured).

There is not much reservation in tf statefiles in git, except, maybe performance while using http backend , but I do have opinions on secure and unsecure configurations being stored in git. Here is my view:
- CI pipeline creates a verified artifacts by Continuously Integrating code from all branches into a branch
- Artifact is created and committed to an artifactory (stealing the sexy jfrog term, but this can be devops artifacts or in a containerized world , registry)
- Deployment is a combination of:
    - static verified (tested, signed etc.) artifact (whether application bits or tf files, whether zip or container)
    - necessary configuration for env where it is being deployed (conn strings, passwords etc)
- CD pipeline is responsible to promote this artifact through various environment life cycles e.g. dev, qa, staging, ephemeral etc. by combining  artifacts with secure and unsecure configuration necessary for the environment being deployed.

![CI and CD](https://raw.githubusercontent.com/vjavle/vjavle.github.io/master/assets/images/deployments.svg)

Given this view, configuration fed to artifact can be classified into 4 categories:

 1. Secure Configuration unchanging based on environment being deployed
 2. Secure Configuration changing based on environment being deployed
 3. UnSecure Configuration unchanging based on environment being deployed
 4. UnSecure Configuration changing based on environment being deployed

Of course, directly or indirectly, all gitops articles will indicate the secrets to be pulled from secure vault. Which is correct and provides prescription for #1 and #2 categories above.
But then it also specifies, other configuration to be committed into git (I am assuming unsecure).

This is where I find issue - By committing unsecure configs for environments into git:
 - configurations can creep into code base (or environment based branches, which is what we try to avoid with Trunk based development as mentioned earlier)
- configurations of ephemeral environments can creep into source control (which are meaningless because of the life time of ephemeral env) or
- conventions (e.g. naming convention) of ephemeral environments can creep into into  code

Ideally , all 4 configurations above should be coming from unsecure and secure config stores (e.g. AppConfig/KeyVault in Azure or parameter store/secrets manager in AWS).
But if the unchanging unsecure configurations are too many, perhaps they can become part of git repo.

**Summary Solution -** System state is stored in combination of git and secure/unsecure config versioned stores.
### 4. Approved changes that can be automatically applied to the system
**Issue #2 -** PR approval being only trigger for new deployment. 

This principle usually means git PR approval. But hence lies the problem, if the solution is applied for issue #1 above. What if the state of a secret changes? One changes password or some encrypted key? How will that configuration be automatically be pushed by a PR approval?

**Summary Solution -**
If gitops suggests "**The entire system described declaratively**" and "**Approved changes that can be automatically applied to the system**" (approval of course, whether manual approval or some automatic super automated testing gates), then CD pipeline should have 2 separate triggers with either/or condition.
  1. Change in latest artifact (or container)
  2. Change in configuration (secure and unsecure)

If one has to follow gitops principle of maintaining state of entire system (hence system can be reproduced with any version - of git as well secure/unsecure config store), it is important to choose a config store which maintains version history of configuration changes as well. Most modern config stores, if not all, do maintain history.
Depending the implementation CD, it is important to link code/bit artifact version with configuration versions to maintain compatibility.

## A note on use of databases and it's code
Now all of this is good. But there remains the problem with the biggest state within the system.

**Business Transaction data**

If your database is code driven (like structure is defined by code , e.g. table and schema definitions), it makes it harder to implement Blue/Green, Canary and Disaster rollback difficult, if not impossible.

God forbid, your DB is not infested with sproc and trigger code.
Coming from 20 years of SQL back ground, it took me slow and steady 3 years to understand, adapt no SQL databases. With hard structure dependent upon codified DDL is gone, rollbacks and canaries are vastly possible.
But that's a whole different conversion i.e. SQL or no SQL db.