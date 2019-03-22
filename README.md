# Taskcluster Retrospectives

Sometimes things go wrong. We don't like it when things go wrong, so let's talk about how to make them go wrong less. This repository contains our retrospectives for events that occur in the operation of Taskcluster.

## Incident Process

 * Incident occurs
 * Someone opens up a google doc, shared with others working on the incident
 * Everyone dumps notes, info, etc. in there
 * Incident is resolved
 * Someone uses the google doc to write up a retrospective document.
   Create a PR in this project with a markdown file that matches the format specified in
   [template.md](https://github.com/taskcluster/taskcluster-retrospectives/blob/master/template.md).
   The file should go in the `retrospectives` directory and should be named `YYYY-MM-DD-${short descriptive title}.md`.
 * That someone flags one or more other participants for review on a PR. The review is to check for accuracy and completeness.
 * The PR gets merged, after any requested changes are made.
 
*NOTE:* there is no need for a meeting for most incidents, unless discussion during the incident or in the review process shows significant disagreement.
