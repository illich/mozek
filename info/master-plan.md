
    Short master plan:


Phase 1: MVP - Context service used via existing chats

    Context service as described in zuri-context-service.md, mostly git based.

    Minimal web ui, for organizing projects, users and connecting google drive documents.

    No chat in this phase.

    Chatting is done on Claude.ai and in Slack via Claude Tag.

    Note: At Zuri, everyone has both Google account and Zoho Mail account. They offer OAuth, so we can use that for authorization.



Phase 2: Full functionality, for Zuri only

    The service developed by us includes the chat as well. For both humans and human-AI interaction.

    Zuri will abandon Slack in this phase.
    But likely still use Claude.ai in paralel.

    Doesn't require support for other companies, payments and some parts can be hard-coded.



Phase 3: Public availability

    Offer this all as a service to 3rd parties.

    Includes a billing part, most likely around $5/user/month. With AI costs extra.