*This draft is a work in progress, and the contents are not final.*


- [Building a TD Assessment Workflow](#Building%20a%20TD%20Assessment%20Workflow)
	- [Signals](#Signals)
	- [Tool Selection](#Tool%20Selection)
	- [Documentation](#Documentation)
	- [Sample workflow](#Sample%20workflow)
- [Running a TD Assessment](#Running%20a%20TD%20Assessment)
	- [Your first technical debt assessment](#Your%20first%20technical%20debt%20assessment)
	- [Iterating on your assessment procedure](#Iterating%20on%20your%20assessment%20procedure)
- [Conclusion](#Conclusion)
* [References](#References)

Technical debt is familiar concept to most professional software developers. It is typically defined as the operational cost associated with making decisions that expedite production of code, with that cost necessarily being repaid at a future date. 

Guides on the topic list many types of technical debt—architecture, infrastructure, test automation—often as many as 13 distinct categories (Alves et al., 2014), each with its own set of indicators and metrics. They discuss complex strategies to compare the time it takes to address technical debt against letting it remain.

The amount of literature on the topic is overwhelming, and little of it provides concrete, actionable guidance. Teams have limited development time and tend to prioritise new features. With such a high opportunity cost to addressing technical debt, it is sometimes deferred until a critical flaw brings work to a standstill.

This article presents the starting point for an **iterative**, **scalable** framework for running technical debt assessments. Following this guide, you will develop an assessment procedure and reporting structure that first targets maintainability of your codebases before addressing systemic issues.

The framework is suitable for for small teams applying the Agile methodology. It assumes that you are developing for a microservice architecture and already have some automated testing in place. A sample stack in Python is presented, but this process should be adaptable to other ecosystems.

## Building a TD Assessment Workflow

Our initial goal is to develop a minimum viable technical debt assessment procedure. Technical debt often misunderstood as "anything a given developer doesn't like about a codebase". Technical debt is not an indicator of poor work, but often a result of rational decisions made by competent developers. While these decisions were sensible in the past, they represent operational risk or cost in the future.

This framework aims to surface major untracked sources of that risk and cost, by first targeting the **maintainability of your repos**. It is almost impossible to refactor your code if it is overly complex, stale, or not covered by quality unit tests. 

This is a documentation-first approach that captures why managing technical debt is important to your particular team, what steps are taken to perform the assessment, and how your findings are structured and reported. Building this documentation has an upfront cost, but should result in repeatable and scalable assessment procedure and a more manageable codebase.

### Signals

We will start by identifying key signals of codebase maintainability. While this is not an exhaustive set of metrics to track technical debt, acting upon these indicators will make your code more manageable and begin to surface errors lurking in the depths of your code. 

This section provides some example thresholds that trigger a corrective action when exceeded. These thresholds are initial suggestions that you can tune as needed.

**Test coverage:** 
Test coverage is the percentage of lines in your codebase covered by automated tests. Strong code coverage is a good indicator of maintainability; if the automated tests covering a function pass before and after refactoring, you can be reasonably assured that you did not introduce unintended changes to its functionality. 

Coverage is not an indicator of that code's quality. Poor unit tests may raise the reported code coverage, but may not test for the functionality that you intended them for.

While it is an indicator of maintainability, high test coverage is not an end in itself. 100% coverage is not desirable as some lines are trivial or serve imported libraries, and should not be tested. 80% is widely considered a reasonable target for code coverage, so use this as your starting threshold. If your builds are reporting less than 80% test coverage, create an issue to write some unit tests.

With the caveats mentioned, measuring test coverage alone can be of limited use. Going beyond this guide, you may choose to track differential test coverage, with a low score resulting in a failed CI/CD build. Differential coverage means measuring tests on new or changed lines of code, and prevents additional debt from accumulating by merging in untested code. 

**Cyclomatic complexity x churn:**
Ease of working with code is quantified with a Maintainability Index, a score that is measured with static code analysis and is made up of several component metrics. One of these metrics is cyclomatic complexity, which measures the number of independent paths through a piece of code. 

As the number of branching `if` statements in a piece of code increases, it becomes more and more difficult for developers to mentally parse and anticipate its runtime behaviour. This difficulty costs developers time when modifying this code, which makes cyclomatic complexity a metric worth tracking.

Tracking cyclomatic complexity alone is not sufficient for prioritising work as part of a technical debt assessment. Nicolas Carlo has an [excellent article](https://understandlegacycode.com/blog/focus-refactoring-with-hotspots-analysis/) on using this metric to complexity hotspots to identify candidates for refactoring. In short, cyclomatic complexity can be mapped against the frequency that files are updated to identify hotspots with high complexity and high churn. These outliers represent portions of your code that are frequently updated and difficult to understand. Nicolas demonstrates exactly how to graph this relationship, and I recommend following this approach in your assessments.

No sample thresholds are provided here, since complexity and the frequency that code is touched will vary widely between teams and projects. Once you begin to graph complexity and churn, you will begin to find outliers and obtain and indicative threshold from there.

It's worth noting that cyclomatic complexity does not tell the complete story, hence why it is only one component of the standard Maintainability Index. Since it maps directly to churn and the developer's ability to understand and maintain code, it is a good starting point for the purpose of this guide.

**Staleness:** 
Staleness is how we measure dependency debt. Often, we are reluctant to upgrade the version of programming language we are using, or move on from a deprecated library that we are relying on. While we avoid the time spent performing migrations now, the time it takes to upgrade will only increase in the future. This makes code harder to maintain as time goes on.

Audit your Docker images, programming languages, and imported libraries or modules. The end-of-life status for many technologies can be found at [endoflife.date](https://endoflife.date/). The release status of libraries can be retrieved from their repositories or documentation. Ensure that Docker images can be rebuilt without issue.

Track staleness using a traffic light system, where green indicates no problems, yellow indicates some staleness, and red triggers an immediate action. If more than 25% of libraries return yellow status, that triggers an upgrade for the entire repo. Some example thresholds have been provided in the table below.

|                          | **Green**                                                | **Yellow**                                                 | **Red**                                                                                      |
| ------------------------ | -------------------------------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| **Python**               | Current LTS/supported minor version                      | ≤12 months from end-of-life                                | Past end-of-life                                                                             |
| **Library dependencies** | ≤1 major version behind _or_ last release ≤18 months old | 2 major versions behind _or_ last release 18-36 months old | ≥3 major versions behind _or_ last release >36 months old _or_ project archived/unmaintained |
| **Docker images**        | Rebuilt within last 30-60 days                           | 2-6 months stale                                           | >6 months stale                                                                              |

Together, these three signals paint a picture of your code's maintainability: test coverage builds confidence in refactoring without loss of functionality, complexity describes the difficulty of changing existing code, and staleness represents cost of maintenance in the future.
### Tool selection

With our signals identified, we must now select software libraries and applications to use when retrieving metrics. These tools must support our aim of building a simple and repeatable assessment process, and avoid introducing additional debt. Here are some principles you can apply when selecting tools.

**Use the least complex tool for the job:**
Use mature, single-purpose, open source utilities where appropriate. Every additional tool we use introduces dependency debt, which we must minimise. Simple tools are easier to maintain, and suffer less from degradation when not maintained. 

Multi-purpose solutions such as all-in-one static code analysis platforms can overhaul their use requiring migrations, and features you depend on might disappear behind a paid subscription. Proprietary, cloud-based services are susceptible to vendor lock-in, so aim for local and open-source tools.

Prefer tools that work with a dependable input and an interpretable output over machine-learning models or tools that provide abstract maintainability scores. The reader of your technical debt report should be able to gauge the maintainability of your repo at a glance.

**Prefer simple, repeatable, well-documented steps over automation:**
It may be tempting to automate each step of the technical debt assessment using Ansible playbooks or complex CI/CD workflows. Clever automation and orchestration saves a lot of time retrieving metrics and filling a report, in theory. In practice, this level of automation leaves your technical debt process itself vulnerable to automation debt, and creates a brittle workflow that is hard to maintain.

Having your team manually retrieve metrics and review code will help them keep in touch with your codebase, and this familiarity will aid them in maintaining it. Maintainability is the foundation of this framework.

Your team can audit each step of your technical debt procedure as it is performed, and identify when a step no longer serves its original purpose. This allows you to mitigate documentation debt within your assessment procedure.

**Apply these principles using your best judgement:**
These principles are mere guidelines and should be applied judiciously. CI/CD automation is a core practice used by many development teams without overstepping into complex builds. Likewise, proprietary and multi-purpose tools that address your team's specific needs do not need to be discarded. Use these principles as starting points when selecting new tools for your technical debt assessment process.

The **Sample workflow*** section below contains an example Python stack with tools selected with these principles in mind.

### Documentation

This documentation-first approach and will produce two artifacts, an **assessment procedure** and a **report template**. Running a technical debt assessment will be as simple as following the procedure and entering the findings into the template.

Documenting your technical debt assessment process will make it repeatable and scalable over time. While there is an upfront cost with building this workflow, a strong procedure and reporting structure will drastically reduce the friction in assessing many repos over multiple instances. 

**Assessment procedure:**
Write the procedure as a set of concrete steps to perform the assessment. Each step should represent a single action, such as running a command in the terminal. Keep the procedure concise and sequential. For each of your tracked signals, an entry in the procedure can be structured as follows:

1) **Signal:** *Which metric is obtained in this task?*
2) **Steps**: *What are the precise steps to obtain this metric?*
3) **Threshold**: *What result will trigger an action?*
4) **Action**: *What actions are recommended as a result?*

%%Screenshot 1%%

**Report template:**
Create a GitHub issue template to form the basis of your reports. This template will keep your reports consistent and make it easy to record your findings. 

Your report template will have the same format as the procedure. Give each signal its own heading, and link these headings to the relevant section in the procedure. Explicit linkages between the report and the procedure will allow your practitioners and reviewers to track what steps were taken and why. 

Each signal will have:

1) **Steps:** Track each step in the procedure with a checklist. Steps that are not relevant to a given assessment can be removed by the practitioner when creating a new report from the template.
2) **Findings**: Add a section for your recorded metrics, and include whether each passes or fails its threshold for action. Findings can often be represented with a table.
3) **Actions**: When your findings trigger an action such as raising code coverage or updating dependencies, link the created issues here. In this way, the report can be used to track progress towards addressing technical debt after the report is complete.

%% Screenshot 2%%

When running a technical debt assessment, create a new report from the template for each repository under review. Have another member of the team review the report to ensure accuracy and build familiarity with different codebases across the team. Done correctly, a technical debt assessment report is a strong piece of documentation in its own right.

### Sample workflow
*(WIP) This subsection will provide a list of tools to track the identified signals, walk you through a sample procedure, and contain screenshots of an example procedure and report.*

## Running a TD Assessment

By this point, you should have a draft of an assessment procedure and a report template. This procedure will take your practitioners through the assessment step by step, detailing how each metric is retrieved and with which tool. 

Your procedure is a living document, designed to be iterated upon. It is unlikely to be optimal from the outset, and will be improved with continuous feedback as you run assessments. This section will take you through your first technical debt assessment and beyond.
### Your first technical debt assessment

Within an Agile team, managing technical debt is a collaborative process and a shared responsibility. Set aside some time in your sprint planning meeting to cover the assessment as a team. Giving each member visibility over the process will allow them to highlight pain-points encountered during development.

Start by identifying and listing all codebases that are in active development. Each codebase should be assessed by the developer most familiar with it.

After completing an assessment, have another developer review the report. This may be a team member who could benefit from increased familiarity with that codebase. Once the findings are accepted as complete, have the practitioner create an issue for each action triggered by the thresholds. Using a "technical debt" tag for each issue and linking them in the report allows you to track your progress in the style of an epic issue.

Larger teams with more complex projects may benefit from time-tracking metrics and other tools to prioritise created issues. For the purpose of this guide, prioritising technical debt tickets will be simple: Assign technical debt issues during planning according to the perceived urgency of the work. This may be based on hotspots that are known to cause difficulty during development, or outliers that scored highly against the initial thresholds. 

**Audit your assessment procedure during the trial run**
Use the initial run to validate the assessment procedure. Some questions you might ask yourself are:

* Is the procedure clearly-written and easy to understand?
* Are the steps for retrieving each metric detailed accurately?
* Can each metric be retrieved in a timely manner?
* Are known pain-points in the codebase getting flagged using the initial thresholds?
* Are the findings actionable?

Seek out feedback from other developers as the maintainer of the procedure. They should find that it guided them quickly through the process without additional effort or guesswork. This can be done during group retrospective meetings.

Consider reworking sections of the procedure if any friction was encountered. Your team will likely opt to let technical debt accumulate if the process is not simple and repeatable.

### Iterating on your assessment procedure

If you have followed this guide to this point, you will have an assessment procedure that audits your codebases for maintainability. This is only the starting point for addressing technical debt, and your documentation system will support the addition of new metrics and processes. 

From here, you will need to decide how often to run a technical debt assessment and how to iterate on your existing procedure. These will vary by team, and even by project within teams. 

For frequency, consider running an assessment every quarter as a starting point. Assessments getting scheduled but not completed may indicate process fatigue, and assessment frequency should be reduced accordingly. Conversely, if your team is routinely encountering friction in the course of development, it may be worth moving the next assessment forward. There is no right answer, all of this is a judgement call.

Likewise, identifying new signals to monitor will require judgement. Security audits are essential for almost all applications. Code quality, delivery friction, integration risk, and monolithic architectures are common sources of debt that are not covered within this guide.

Always maintain awareness of your codebase. If your code is no easier to maintain after a round of technical debt, then you may be monitoring the wrong signals or need to adjust your thresholds. Failing to close tickets that were created to address technical debt can also indicate an inflated assessment scope.

This guide presented a lightweight assessment procedure for small and medium teams and projects, so time-tracking and debt prioritisation strategies were discussed informally. As your team and projects grow, consider [formalising these strategies](https://flagshark.com/blog/technical-debt-prioritization-framework/).

With so many ways to iterate on your assessment procedure, be sure not to change too much at once. For example, adding one new metric and running a round of assessments will allow you to gather clear feedback on the effectiveness of that change. That way, you have the basis for a robust assessment procedure that you can build upon well into the future.

## Conclusion
*(WIP) This section will summarise the main points from this article.*

## References

Alves, N.S.R., Ribeiro, L.F., Caires, V., Mendes, T.S. and Spinola, R.O. (2014). Towards an ontology of terms on technical debt. _2014 Sixth International Workshop on Managing Technical Debt_. doi:10.1109/mtd.2014.9.

Carlo, N. (n.d.). _Focus refactoring on what matters with hotspots analysis - change messy software without breaking it_. [online] Understand Legacy Code. Available at: https://understandlegacycode.com//blog/focus-refactoring-with-hotspots-analysis [Accessed 9 Nov. 2026].

endoflife.date. (n.d.). _Home_. [online] Available at: https://endoflife.date/ [Accessed 9 Nov. 2026].

McGrath, J. (2026). _Technical debt prioritization framework_. [online] FlagShark Blog. Available at: https://flagshark.com/blog/technical-debt-prioritization-framework/ [Accessed 9 Nov. 2026].
