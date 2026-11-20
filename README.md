# box-for-snc-custom-embed

A ServiceNow scoped application ("Box Embed Demo") that embeds Box content in a ServiceNow UI Builder page. It is a sibling extension to the **Box Integration** store app (`x_bonh_box`): the custom work lives in its own scope so it survives vendor upgrades, and it talks to the vendor app only through that app's public subflows.

> [!WARNING]
> **Disclaimer: community example, not production code.**
> This is an open source community example, shared for learning and demonstration. It is not an official Box or ServiceNow product, it is not supported by either company, and it is not meant to be used as production code. It has not been through a security review, performance testing, or the hardening a production deployment needs. Use it at your own risk, review it against your own standards, and test it in a non-production instance first. It is provided "as is" under the [MIT License](LICENSE), without warranty of any kind.

## What it does

The app builds a Box Embed URL on the server and hands it to an iFrame on a UI Builder page. One script include does the work, and a transform data resource exposes it to UI Builder.

Supported item types:

| `itemType` | What is embedded |
| --- | --- |
| `folder` | A Box folder |
| `file` | A single Box file |
| `hubs` | A Box Hub |
| `app` | A Box App |
| `ai-chat` | The Box Hubs AI chat for a hub |

## How it works

1. A UI Builder page adds the **Box Embed Demo** data resource and passes it `itemType`, `itemId`, and optionally `sharedLink`.
2. The data resource calls `BoxEmbedService.build(itemType, itemId, source, sharedLink)`.
3. `BoxEmbedService` validates the inputs, then runs the vendor subflow `x_bonh_box.access_gate` to confirm the current user is allowed to see Box content.
4. It reads the embed base URL from the vendor system property `x_bonh_box.embed_base_url` and builds the embed URL for the item type.
5. It runs the vendor subflow `x_bonh_box.mint_login_token` and, when a token comes back, adds it to the URL so the user is signed in to the embed.
6. The page binds the returned URL to the `src` of an iFrame component.

Nothing sensitive is stored in this app. The base URL comes from a system property, and the login token is created at runtime for each request and is never saved or logged.

## Repository layout

ServiceNow source control writes the application to the repository as XML, one file per application record. The app files below appear after the first commit from ServiceNow Studio.

```
box-for-snc-custom-embed/
├── README.md                      # This file
├── LICENSE                        # MIT License
├── .gitignore                     # Keeps local, OS, editor and secret files out of the repo
├── sys_app_<sys_id>.xml           # The application record: name, scope (x_637605_box_ext_0), version
├── checksum.txt                   # Written by ServiceNow to detect changes between commits
├── update/                        # The application files, one XML file per record
│   ├── sys_script_include_<sys_id>.xml            # BoxEmbedService: validates input, checks access, builds the embed URL
│   ├── sys_ux_data_broker_transform_<sys_id>.xml  # "Box Embed Demo" data resource that UI Builder pages call
│   ├── sys_security_acl_<sys_id>.xml              # Execute ACL on the data resource
│   ├── sys_security_acl_role_<sys_id>.xml         # Grants that ACL to the x_bonh_box.box_user role
│   └── sys_scope_privilege_<sys_id>.xml           # Cross-scope privileges (one file each, see below)
└── author_elective_update/        # Deletion records and other elective updates tracked by ServiceNow
```

Do not edit the XML files by hand. Change the app in ServiceNow Studio and commit from there, so the files and `checksum.txt` stay in step.

### Cross-scope privileges

The app requests these privileges so it can call into the vendor scope and the Flow API:

- `ScriptableFlowRunner` (`subflow`, `inForeground`, `withInputs`, `run`) and `ScriptableFlowRunnerResult.getOutputs`, to run the vendor subflows
- Glide API: properties, to read the vendor system properties
- Glide API: user roles and groups, to check the user's role

## Requirements

- A ServiceNow instance with UI Builder
- The **Box Integration** store app (`x_bonh_box`) installed and connected to your Box enterprise
- These items from the vendor app:
  - Subflows `x_bonh_box.access_gate` and `x_bonh_box.mint_login_token`
  - System property `x_bonh_box.embed_base_url`, set to your Box embed URL (for example `https://acme.app.box.com/embed`)
  - System property `x_bonh_box.embed_ui_folder_navigation`
  - Role `x_bonh_box.box_user`
- Users who have the `x_bonh_box.box_user` role and a connected Box account

## Install

1. In ServiceNow Studio, choose **Import from source control**.
2. Enter this repository's URL and your Git credentials.
3. Once the import finishes, open the **Box Embed Demo** application and approve the cross-scope privileges if your instance asks.
4. Check that the vendor system properties listed above are set.

## Use it on a page

1. Open your page in UI Builder.
2. Add a data resource and pick **Box Embed Demo**.
3. Set its inputs:
   - `itemType` (required): one of `folder`, `file`, `hubs`, `app`, `ai-chat`. Defaults to `folder`.
   - `itemId`: the Box ID of the folder, file, hub or app.
   - `sharedLink` (optional): a Box shared link value for the item.
4. Add an iFrame component and bind its `src` to the data resource's output.

Box IDs are specific to your Box enterprise. Replace any IDs from a demo page with your own.

## Things to know

- **Third-party cookies.** The embed runs in an iFrame on a different domain. Browsers that block third-party cookies will stop it from loading until cookies are allowed for Box.
- **Multi-factor authentication.** The first time a user opens an embed in a new browser, Box may ask for an MFA challenge before the content appears.
- **Login token in the URL.** The login token is passed as a URL parameter. Treat the built URL as sensitive: do not log it or write it to a record.
- **Errors.** When access is denied, a property is missing, or a subflow fails, the service returns a message for the user and logs the technical detail to the system log.

## License

Released under the [MIT License](LICENSE).
