# ctf.json
The `ctf.json` is the central configuration file, used by the EnoEngine. It MUST be named `ctf.json` and MUST be located in the working directory from which the EnoEngine is started.

## Format
```ts
interface ctfjson {
    title: string;
    flagValidityInRounds: number;
    checkedRoundsPerRound: number;
    roundLengthInSeconds: number;
    dnsSuffix: string;
    teamSubnetBytesLength: number;
    flagSigningKey: string;
    encoding: string;
    services: Service[];
    teams: Team[];
}
interface Service {
    id: number;
    name: string;
    flagsPerRoundMultiplier: number;
    noisesPerRoundMultiplier: number;
    havocsPerRoundMultiplier: number;
    weightFactor: number;
    active?: boolean;
    checkers: string[];
}

interface Team {
    id: number;
    name: string;
    address: string | null;
    teamSubnet: string;
    logoUrl: string | null;
    countryCode: string | null;
    active?: boolean;
}
```

The file must be in JSON format and expects an object conforming to the `ctfjson` interface at the root level, for example:
```json
{
    "title": "title",
    "flagValidityInRounds": 5,
    ...
}
```

### ctfjson
#### title
The name/title of the CTF, this values is unused and is expected to be removed from the specification.
#### flagValidityInRounds
This specifies how many rounds flags remain valid from the round in which they were "created" and placed by putflag. The value `0` means the flag is only valid in the round in which it was put, `1` means it will also be valid in the next round and so on.
#### checkedRoundsPerRound
For how many of the most recent `putflags` from previous rounds `getflag` will be called. The value `1` means `getflag` will only be called for the `putflag` which happened in the same round, `2` means it will also be executed for the `putflag` from the previous round and so on.
#### roundLengthInSeconds
The length of one round in seconds. Note that each round is divided into four quarters and each task only has roughly one quarter of a round minus a safety margin to complete. This means for a round length of `60` seconds, each checker task has less than `15` seconds to complete.
#### dnsSuffix
When no address (IP or domain name) is specified for a team, the DNS suffix will be used to create an address automatically following the scheme `team<teamId><dnsSuffix>`. This means the dnsSuffix MUST include a leading dot, i.e. setting the dnsSuffix to `.example.com` results in addresses looking like `team1.example.com`.

This dnsSuffix will also be added to the `scoreboardInfo.json`.
#### teamSubnetBytesLength
The length of the network mask in bytes, ASSUMING AN IPv6 ADDRESS. When using IPv4 subnets, the subnets in the `Team` interface (see below) must be specified as IPv4-mapped IPv6 addresses, which are located in the `::ffff:0:0/96` network and can be noted like `::ffff:192.168.1.0`. Note that the four bytes of the IPv4 address are delimited by `.` and not `:`, which means they are not hexadecimal values in 2-byte-blocks like in a usual IPv6 address.

When working with IPv4 subnets with a `/32` mask, the correct `teamSubnetBytesLength` is `16`. When working with IPv4 subnets with a `/24` mask, the correct `teamSubnetBytesLength` is `15`.
#### flagSigningKey
The key which is used to sign the flags. This MUST remain secret during a CTF, since knowing the `flagSigningKey` allows generating flags which will be considered correct by the EnoEngine. This can be any arbitrary string and does not need to conform to any length/format requirements.
#### encoding
The encoding of the flags. Currently this supports two formats, namely `Legacy` and `UTF8`. In legacy mode, the flags are using only ASCII characters and adhere to the regular expression "`ENO[a-zA-Z0-9+\/]{48}`". In UTF8 mode, the flags contain unicode characters and adhere to the regular expression "`🏳️‍🌈\X{4}`", where `\X` matches any number of Unicode characters that form an extended Unicode sequence. Please refer to [regex101.com](https://regex101.com) or any relevant specification for more details.
#### services
A JSON array containing service objects, see below.
#### teams
A JSON array containing team objects, see below.
### Service
#### id
This ID must be unique for each service during a CTF and must be greater than (not equal to) `0`. It MUST fit into an `int` variable in C#. It is recommended to use ascending IDs starting with `1`.
#### name
The name of the service which will appear in log messages and is included in the `scoreboardInfo.json`.
#### flagsPerRoundMultiplier
When the multiplier is `1`, each service gets as many `putflag`s per round as it supports `flagVariants` (see checker specification for more details). When setting to multiplier to `2`, there will be two `putflag`s per `variantId` each round and so on.
#### noisesPerRoundMultiplier
Same as `flagsPerRoundMultiplier`, just for `putnoise` and `noiseVariants`.
#### havocsPerRoundMultiplier
Same as `flagsPerRoundMultiplier`, just for `havoc` and `havocVariants`.
#### weightFactor
Not yet implemented, subject to be removed.
#### active
When set to `false`, the service is temporarily disabled and behaves as if it was not part of the config at all, meaning it will not receive any new tasks. This is only used if a service should be temporarily disabled during a CTF, in which case a restart of the Engine is required. Defaults to `true`.
#### checkers
An array of strings, each containing the URL of one checker. For example: `["http://192.168.1.7:9002"]`. The EnoLauncher does round robin load balancing when more than one address is specified, it does NOT support fail-over if one of the checkers is down. For more advanced load balancing, it is recommended to use a dedicated load balancer and specifying its address as a single entry in this array.
### Team
#### id
This ID must be unique for each team during a CTF and must be greater than (not equal to) `0`. When no address is specified, this ID is used to construct the address based on the `dnsSuffix` (see above).
#### name
The name of the team, this appears in log messages and is included in the `scoreboardInfo.json`.
#### address
The address of the team, this is passed as service addres to the checkers. See checker protocol specification for more details.
#### teamSubnet
The subnet of the team. This, together with the `teamSubnetBytesLength`, is used by the EnoFlagSink to determine which team an incoming flag submission connection belongs to.
#### logoUrl
The complete URL to an image file containing the logo for the team. This is included in the `scoreboardInfo.json`.
#### countryCode
The ISO 3166-1 alpha-2 country code in uppercase letters, for example `"DE"` for germany. This is included in the `scoreboardInfo.json`.
#### active
When set to `false`, the team is temporarily disabled and behaves as if it was not part of the config at all, meaning it will not receive any new tasks. This is only used if a team should be temporarily disabled during a CTF, in which case a restart of the Engine is required. Defaults to `true`