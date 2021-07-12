# Mapping to/from C8Y

## At Mapper Start-Up

### Send Thin Edge JSON Software List Request

Outgoing to `tedge/inbound/software/list`.

```json
{
    "id": 123
}
```

The mapper must generate a unique ID. Refer to <Link to Albin's doc later>.

### From Thin Edge JSON Software List Response to c8y_SoftwareList(116) [Discussion required]

Incoming to `tedge/outbound/software/list`.

```json
{
    "id": 123,
    "status": "SUCCESSFUL",
    "list": [
        {
            "type": "debian",
            "list": [
                {
                    "name": "nodered",
                    "version": "1.0.0"
                },
                {
                    "name": "collectd",
                    "version": "5.7"
                }
            ]
        },
        {
            "type": "docker",
            "list": [
                {
                    "name": "nginx",
                    "version": "1.21.0"
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6"
                }
            ]
        }
    ]
}
```

Outgoing to `c8y/s/us`.

We need to make a design decision here.\
Since C8Y doesn't have `type` field, should we have to inform `type` to c8y? If yes, how?

1) Ignore `type` completely

```
116,nodered,1.0.0,,collectd,5.7,,nginx,1.21.0,,mongodb,4.4.6,
```

- [+] Easy way to go. We can say that it's a design problem of C8Y Cloud.
- [--] User can't know which type of `nginx` for example. Can it be debian, snap, or docker...

2) Add `type` as a suffix of package name

```
116,nodered:debian,1.0.0,,collectd:debian,5.7,,nginx:docker,1.21.0,,mongodb:docker,4.4.6,
```

- [+] User can know which type of package.
- [-] If user want to install a new version of package, we have to force user to name `<software>:<type>` in C8Y's software repository.
- [-] It forces duplicated work on user, along with Proposal in [How to know that we want to install from the standard repository](#how-to-know-that-we-want-to-install-from-the-standard-repository-eg-apt-install-collectd).

## Runtime

### From Thin Edge JSON Get PENDING SoftwareUpdate operations Request to GET PENDING operations (500)

Incoming in: `where?`

```json
{
    "unknown": "no spec yet available"
}
```

Outgoing to: `c8y/s/us`

```
500
```

### From SoftwareUpdate operation (528) to Thin Edge JSON Software Update Request [Discussion required]

Incoming in: `c8y/s/ds`

We need to make a decision here. There are two problems to think about.
1. How can mapper know `type`?
2. If user wants to install `collectd` from standard apt repository, how to know it?

#### How to determine the type?

Proposal 1: Get to know the type from URLs. Anticipate the type from the file extensions in URLs.\
It is aligned with proposal 1) in [Software List Response](#from-thin-edge-json-software-list-response-to-c8y_softwarelist116-discussion-required).

```
528,external_id,nodered,1.0.0,url1,install,collectd,5.7,url2,install,nginx,1.21.0,url3,install,mongodb,4.4.6,url4,delete
```

Proposal 2: Enforce user to name software in the format `<software>:<type>`\
It is aligned with proposal 2) in [Software List Response](#from-thin-edge-json-software-list-response-to-c8y_softwarelist116-discussion-required).

```
528,external_id,nodered:debian,1.0.0,url1,install,collectd:debian,5.7,url2,install,nginx:docker,1.21.0,url3,install,mongodb:docker,4.4.6,url4,delete
```

#### How to know that we want to install from the standard repository (e.g. apt install collectd)?

Proposal A: Enforce user to use `tedge://<type>` as URL. e.g. `tedge://debian`, `tedge://docker`...

```
528,external_id,nodered,1.0.0,tedge://debian,install,collectd,5.7,tedge://debian,install,nginx,1.21.0,tedge://docker,install,mongodb,4.4.6,tedge://docker,delete
```

Proposal B: "A non-valid formatted URL (or missing URL) is interpreted as "install from repo".
that includes: entering " " (space) as URL.

```
528,external_id,nodered,1.0.0,,install,collectd,5.7,,install,nginx,1.21.0,,install,mongodb,4.4.6, ,delete
```

#### Combination summary how SmartREST 528 looks

Proposal 1 & A:

```
528,external_id,nodered,1.0.0,tedge://debian,install,collectd,5.7,https://collectd.org/download/collectd-tarballs/collectd-5.12.0.deb,install,nginx,1.21.0,tedge://docker,install,mongodb,4.4.6,tedge://docker,delete
```

Proposal 2 & B:

```
528,external_id,nodered:debian,1.0.0, ,install,collectd:debian,5.7,https://collectd.org/download/collectd-tarballs/collectd-5.12.0.deb,install,nginx:docker,1.21.0, ,install,mongodb:docker,4.4.6, ,delete
```

Then, outgoing to: `tedge/inbound/software/update`

```json
{
    "software-update": [
        {
            "type": "debian",
            "list": [
                {
                    "name": "nodered",
                    "version": "1.0.0",
                    "action": "install"
                },
                {
                    "name": "collectd",
                    "version": "5.7",
                    "url": "https://collectd.org/download/collectd-tarballs/collectd-5.12.0.deb",
                    "action": "install"
                }
            ]
        },
        {
            "type": "docker",
            "list": [
                {
                    "name": "nginx",
                    "version": "1.21.0",
                    "action": "install"
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6",
                    "action": "remove"
                }
            ]
        }
    ]
}
```

**Attention**:
- Uninstallation terminologies are different in C8Y and Thin Edge JSON. C8Y uses `delete`, although Thin Edge JSON uses `remove`. 
- The `type` assumption is possible to fail.

### From Thin Edge JSON Software Update Executing to Operation Update EXECUTING (501)

Incoming to `tedge/outbound/software/update`.

```json
{
    "status": "EXECUTING"
}
```

Outgoing to `c8y/s/us`.

```
501,c8y_SoftwareUpdate
```

### From Thin Edge JSON Software List Response to c8y_SoftwareList(116) and Operation Update SUCCESSFUL (503)

Incoming to `tedge/outbound/software/list`.

```json
{
    "id": 123,
    "status": "SUCCESSFUL",
    "list": [
        {
            "type": "debian",
            "list": [
                {
                    "name": "nodered",
                    "version": "1.0.0"
                },
                {
                    "name": "collectd",
                    "version": "5.7"
                }
            ]
        },
        {
            "type": "docker",
            "list": [
                {
                    "name": "nginx",
                    "version": "1.21.0"
                },
                {
                    "name": "mongodb",
                    "version": "4.4.6"
                }
            ]
        }
    ]
}
```

Outgoing to `c8y/s/us` with two messages.

The first message is SmartREST `116`, that is `c8y_SoftwareList`.
Refer to [From Thin Edge JSON Software List Responce to c8y_SoftwareList (116)](#from-thin-edge-json-software-list-response-to-c8y_softwarelist116-discussion-required).

The second message is operation update to `SUCCESSFUL`.
```
503,c8y_SoftwareUpdate
```

### From Thin Edge JSON Software List Response to c8y_SoftwareList(116) and Operation Update FAILED (502)

Incoming to `tedge/outbound/software/list`.

```json
{
    "status":"FAILED",
    "reason":"Partial failure: Couldn't install collectd and nginx",
    "current-software-list": [
        {
            "type": "debian",
            "list": [
                {
                    "name": "nodered",
                    "version": "1.0.0"
                }
            ]
        },
        {
            "type": "docker",
            "list": [
                {
                    "name": "mongodb",
                    "version": "4.4.6"
                }
            ]
        }
    ]
}
```

Outgoing to `c8y/s/us` with two messages.

The first message is SmartREST `116`, that is `c8y_SoftwareList`.
Refer to [From Thin Edge JSON Software List Responce to c8y_SoftwareList (116)](#from-thin-edge-json-software-list-response-to-c8y_softwarelist116-discussion-required).

The second message is operation update to `FAILED` with the given reason.

```
502,c8y_SoftwareUpdate,"Partial failure: Couldn't install collectd and nginx"
```

## From Thin Edge JSON device capabilities to c8y_SupportedOperations(114) [Under the discussion]

Incoming to `where?`

```json
{
  "unknown": "no spec yet available"
}
```

Outgoing to `c8y/s/us`.

```
114,c8y_SoftwareUpdate,c8y_Restart
```
