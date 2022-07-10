# Checker Protocol v2
The checker protocol is the HTTP-based protocol used by the engine to communicate with the checkers.

The checker is expected to respond with an HTTP status code 200 to the following requests:

## `GET /service`
Response:
```ts
interface CheckerInfoMessage {
    serviceName: string;
    flagVariants: number;
    noiseVariants: number;
    havocVariants: number;
    exploitVariants: number;
}
```
### CheckerInfoMessage
#### serviceName
The name of the service as reported by the checker.
#### flagVariants
The number of different flag variants supported and requested by the checker/service. Each variant MUST correspond to a different location in the service where the flag is stored (flag store).
#### noiseVariants
The number of different noise variants supported and request by the checker/service. Different variants do not necessarily need to store the noise in different locations. Unlike a flag, the noise does not need to remain secret and is only used during a `getnoise` to check whether a noise is still present in the service, even if it is stored in a publicly accessible location.
#### havocVariants
The number of different havoc variants supported and requested by the checker/service. Each havoc is an independent task, meaning there is no concept of two or more related tasks like a `putflag` or `getflag`.

A havoc can be used to test any functionality in the service which is not covered by `putflag`/`getflag` and `putnoise`/`getnoise`. While a havoc MAY use information from other tasks, it MUST NOT fail if such information is missing from either the checker database or the service. For example, it might try to reuse an account created during `putflag`, but is NOT allowed to fail when this account is missing, unlike the corresponding `getflag`, which is expected to fail when it can no longer login. Note that `havoc` may run at any time during a round, even before `putflag` or `putnoise`.

#### exploitVariants
The number of different exploits. This MUST be greater than or equal to the number of `flagVariants`.

## `POST /`
Parameter:
```ts
interface CheckerTaskMessage {
    taskId: number;
    method: "putflag" | "getflag" | "putnoise" | "getnoise" | "havoc" | "exploit";
    address: string;
    teamId: number;
    teamName: string;
    currentRoundId: number;
    relatedRoundId: number;
    flag?: string | null;
    variantId: number;
    timeout: number;
    roundLength: number;
    taskChainId: string;
    flagRegex?: string | null;
    flagHash?: string | null;
    attackInfo?: string | null;
}
```
Response:
```ts
interface CheckerResultMessage {
    result: "INTERNAL_ERROR" | "OK" | "MUMBLE" | "OFFLINE";
    message: string | null;
    attackInfo?: string | null;
    flag?: string | null;
}
```
### CheckerTaskMessage
#### taskId
This ID MUST be unique for each task during a CTF. It is the responsibility of the caller (usually the Engine) to ensure this criterion is met.
#### method
The method to be executed in this task.

`putflag`:
* A `putflag` MUST store the flag (see the flag parameter below) in the service in such a way that it can be retrieved by a `getflag` at a later point in time from a correctly working service. 
* The flag MUST be stored in a "secret" location in the service, meaning either some form of credentials only known by the checker or an exploit are required to get the flag.
* The `putflag` MUST store any credentials which are required by the `getflag` to retrieve the flag (e.g. username and password used for registering an account) in the checker's database.
* The `putflag` MUST use the `taskChainId` (see below) as identifier in the database to store any credentials required by a later `getflag`, as this is guaranteed to be identical for any subsequent `getflag` requests for the same flag. The `taskChainId` is also guaranteed to be different for tasks in the same round for different teams and tasks for the same team in different rounds.
* The caller MUST NOT call `putflag` with the same `flag` or `taskChainId` twice during a CTF. (Note that supporting such behavior in the checker might still be useful for testing purposes.)
* A `putflag` MAY return a value for `attackInfo` providing some identifying information the attackers may use to identify where the flag is stored in the service, such as a username in a service that doesn't provide other user enumeration capabilities.

`getflag`:
* A `getflag` request MUST check whether the flag (see the flag parameter below) placed during the previous `putflag` with the same `taskChainId` is still present in the service.
* A checker MUST NOT return `"OK"` (see CheckerResultMessage below) unless the flag was found in the service. It MUST return `"MUMBLE"` if the flag was not found or `"OFFLINE"` if the service could not be reached.
* A checker MUST use the `taskChainId` (see below) as identifier to retrieve any credentials stored by the related `putflag`, as this value is guaranteed to be identical to the `taskChainId` during the `putflag`.
* A `getflag` MUST NOT return an `"INTERNAL_ERROR"` (see CheckerResultMessage below), even if the corresponding `putflag` did not return `"OK"`. It could be that the flag was stored successfully but the `putflag` did not return `"OK"` due to some other reason, for example because it timed out before any subsequent actions were completed.
* If required credentials, which should have been stored during the `putflag`, are missing from the database, the checker MUST return `"MUMBLE"` (see CheckerResultMessage below) immediately without performing any subsequent checks.
* A checker MUST be able to handle multiple subsequent `getflag` calls for the same `putflag`/`taskChainId`.

`putnoise`:
* A `putnoise` MUST store some noise generated by the checker to the service in such a way that it can be retrievd by a subsequent `putnoise` with the same `taskChainId` at a later point in time from a correctly working service.
* A checker MUST generate sufficiently unique noise to avoid false positives. Checker implementations might either generate something randomly and store it in the database using the `taskChainId` as identifier, or use the `taskChainId` as a seed to feed a deterministic RNG to be able to generate the same noise during a subsequent `getnoise`.
* The noise MAY be stored in a "public" location in the service, meaning it might be visible to all normal users of the service without any secret credentials or an exploit.
* THe noise MUST NOT be stored in a location in the service where other teams can legally remove it, i.e., without an exploit.
* The checker MUST use the `taskChainId` as the identifier in case any credentials or the noise itself need to be stored in the database for a subsequent `getnoise`.

`getnoise`:
* A `getnoise` MUST check whether the noise placed during a previous `putnoise` is still present in the service.
* The checker MUST NOT return `"OK"` unless the noise was found in the service.
* If required credentials, which sould have been stored during the `putnoise`, are missing from the database, the checker MUST return `"MUMBLE"` immediately without performing any subsequent checks.
* A checker MUST be able to handle multiple subsequent `getnoise` calls for the same `putnoise`/`taskChainId`.

`havoc`:
* A `havoc` MUST NOT depend on any tasks from previous rounds or the same round, including credentials that should have been created during one of these task.
* A `havoc` MAY use e.g. credentials from other tasks, provided it does not fail if these credentials are missing.

`exploit`:
* The `exploit` method is only used during testing and MUST NOT be called during a CTF.
* Each flag store MUST be exploitable by at least one `exploit` variant.
* A flag store MAY be exploitable by more than one `exploit` variant.
* An `exploit` variant MAY be able to retrieve flags from multiple flag stores.
* Each flag store SHOULD have an `exploit` variant that is only able to retrieve flags from that flag store.
* The `exploit` method checks whether it is able to retrieve a flag matching the `flagHash`. If yes, it MUST return `OK` and the found flag as part of the result. If not, it MUST return `MUMBLE`. It is the caller's responsibility to ensure the flag is stored in the expected flag store.
* An `exploit` MAY need to be connected to the service before `putflag` is called. In that case it is the caller's responsibility to ensure `putflag` is called while the `exploit` method is running.

General requirements:
* A checker MUST NOT make any assumptions about the order of the execution of any tasks, except the following:
    * a `getflag` with the same `taskChainId` as the related `putflag` is always executed after the `putflag` is completed
    * a `getnoise` with the same `taskChainId` as the releated `putnoise` is always executed after the `putnoise` is completed
#### address
The address of the target team's vulnbox. Can be either an IP address or a valid hostname.
#### teamId
The id of the target team. Primarily used for logging purposes.
#### teamName
The name of the target team. Primarily used for logging purposes.
#### currentRoundId
The ID of the current round.
#### relatedRoundId
When the method is `getflag` or `getnoise`, the related round ID refers to the round in which the `putflag` or `putnoise` with the same `taskChainId` was executed. For all other methods, it is identical to the `currentRoundId`.

A checker MUST NOT use the `relatedRoundId` as the identifier to store and retrieve data for later rounds or from previous rounds for related `putflag`/`getflag` or `putnoise`/`getnoise` tasks, instead it MUST use the `taskChainId` as the identifier.
#### flag
For `putflag` and `getflag`, this is the flag which is supposed to be stored or retrieved by the task. For all other methods, the flag is not set or `null`.
#### variantId
The variant id for the method, used to support different flag, noise and havoc methods. This id will be a value between 0 and `flagVariants - 1` (see CheckerInfoMessage above) for `putflag`/`getflag`, the same for noise and havoc with `noiseVariants` and `havocVariants` respectively.

A checker MUST return an `"INTERNAL_ERROR"` when called with a `variantId` it does not support. (This is required for testing automatically whether the specified number of variants matches the implementations in the methods.)
#### timeout
The timeout for the task in milliseconds. If a checker does not return a response within the timeout, the caller assumes that the result is `"OFFLINE"`.

A checker or checker library SHOULD enforce the timeout internally to avoid consuming computational resources after the caller has cancelled the request.
#### roundLength
The length of a round in milliseconds.
#### taskChainId
The unique identifier of a chain of tasks (i.e. `putflag` and `getflag`s or `putnoise` and `getnoise` for the same flag/noise share an Id, each havoc has its own Id). It is up to the caller to ensure the aforementioned criteria are met, the Engine achieves this by composing it the following way: `"{flag|noise|havoc}_s{serviceId}_r{relatedRoundId}_t{teamId}_i{uniqueVariantIndex}"`.

The `taskChainId` MUST be used as the identifier in the database in case e.g. credentials created during `putflag` need to be stored and are required in a subsequent `getflag`.

A checker MUST support being called multiple times with the same `method`, `serviceId`, `roundId`, `teamId` and `variantId`, in which case the `uniqueVariantIndex` can be used to distinguish the taskChains.

#### flagRegex
For the `exploit` method, this is a regular expression that the flag to be exploited must match. For all other methods, the `flagRegex` is not set or `null`.

#### flagHash
For the `exploit` method, this is the hex-encoded SHA256 hash of the flag to be searched by the exploit. For all other methods, the `flagHash` is not set or `null`.

#### attackInfo
For the `exploit` method, this is the `attackInfo` that was returned by the `putflag` method for the flag to be found, if any was returned, and not set or `null` otherwise. For all other methods, the `attackInfo` is not set or `null`.

### CheckerResultMessage
#### result
The result of the task. For the purpose of scoring, there is no difference between `"MUMBLE"` and `"OFFLINE"` and the distinction is only made for informational purposes.

In general, `"OFFLINE"` should be returned when an issue with the connection to the service occurs, `"MUMBLE"` should be returned in all other failure cases, for example when the service sends an unexpected response. An `"INTERNAL_ERROR"` MUST NOT be returned when the checker is called within the specification, most notably when all of the aforementioned criteria are met.

When the service behaves as expected, the checker MUST return `"OK"`.
#### message
A message describing the error that occured, this is displayed on the public scoreboard if it is not null.

The message MUST be `null` when the `result` is `"OK"`.

The message SHOULD provide additional information on failure cases.

The checker MUST NOT include internal details in this message, most notably it MUST NOT include the flag or other secret information.

When the `result` is `"INTERNAL_ERROR"`, the message MUST NOT be displayed publicly to participants and MAY contain secret information.
#### attackInfo
For results from `putflag`, this is an arbitrary string that will be publicly displayed for each team and round if it is not `null`. For all other methods, this field must be unset or `null`.

It SHOULD provide attackers with otherwise unavailable information required to mount an exploit retrieving this flag, such as a username.

#### flag
For results from `exploit`, if the result is `"OK"`, this MUST be the flag matching the `flagHash`. For other methods or when the result is not `"OK"`, this must be unset or `null`.
