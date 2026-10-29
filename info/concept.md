
For our company, https://zuri.com/ I need a system for productive use of AI over our assets.

Our current setup:

    Everyone (30 people) has access to:
    - Google Drive
    - Slack
    - Trello
    - Claude (Team)

    We can switch Slack or Trello easily. Google Drive and Claude will mostly stay.

Structure

    We have teams and projects (usually a project has 4-5 core members, but wider access up to basically everyone for some projects is expected).

    Not much projects - 2 big ones, and ~10 smaller ad hoc projects. Everything is pretty connected, the projects are not much contained. For example Ground station serves for both the small drones and big demonstrator. There are lots of small drones. Flight control is also mostly shared. Etc.


What we need

    For each project, very easy way how to connect:

    - for AI, particular documents/spreadsheets from Google Drive to the project
    (we don't really want to give a broad read-everything access to AI - because we have quite a mess including older and obsolete documents, and also for security principles)
    most likely for each project, AI should have access to 10-100 hand-picked documents

    - each document should have one person as the maintainer - mostly because AI should "complain" to particular person if it founds inconsistencies there

    - generally, the documents will be read-only for AI, but later, even write access will be needed

    - each project has people - access levels: admin (can add people and connect documents), member (read/write access, proposes document additions), reader (read-only)

    - people and AI can chat about anything, AI has access to all the project documents; the chats can be personal (just 1 human and AI), team (humans and AI) or custom groups

    - generally, AI knows everything, with the exception of "whisper" chats, usually just two humans

    - for AI, we assume we will use Claude (Opus 5.5 now in particular) for everything, but it would be nice to have ability to connect to any AI, including local ones

    - the conversations including AI should have the same features as regular Claude chat - adding custom attachments, have a long thread of back and forth conversation

    - the UX should be quite friendly - with clear visual clues about what project and we talking about, what is the visibility, topic, etc.

    - it will replace Slack for us too - but that is not the goal, the primary is AI chat over the curated set of documents from Google Drive

    - for chats with custom groups, people can be invited even later, prefeably with a choice of visibility (from beginning, from now)

    - the service should keep .md versions of all the google files and update them automatically when the source changes; the diffs should be kept internally for a while, to help with the next step (updated summaries)

    - also the service should maintain per-project summaries - one .md which tries to summarize all the files it has access to; rebuild on every major change of source files

    - preferably, for AI chats we should utilize the Team subscription, so not bill according to API prices