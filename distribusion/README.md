[![CircleCI](https://circleci.com/gh/miry/samples/tree/master.svg?style=svg)](https://circleci.com/gh/miry/samples/tree/master)

Implement library to parse data from the service and upload in unified format.

## `GET /routes`

### Params

- `passphrase` - string.
- `source` - string. possible values: sentinels, sniffers, loopholes

### Response

Return a zip file

```
HTTP/1.1 200 OK
Content-Disposition: attachment; filename*=UTF-8''sentinels
Content-Length: 996
Content-Type: application/zip
....
```

## `POST /routes`

Except only query params

### Params

- `passphrase` - string.
- `source` - string. possible values: sentinels, sniffers, loopholes
- `start_node` - alphanumeric code. values: alpha, beta, gamma, delta, theta, lambda, tau, psi, omega
- `end_node` - alphanumeric code. values: alpha, beta, gamma, delta, theta, lambda, tau, psi, omega
- `start_time` - ISO 8601 UTC time. YYYY-MM-DDThh:mm:ss
- `end_time` - ISO 8601 UTC time. YYYY-MM-DDThh:mm:ss

### Response

- 201 - very lucky to create
- 401 - if missing passphrase
- 503 - some field is not correct

USAGE
=====

To run localy application.

```
$ bin/local -p <passphrase> # Just run sync
$ bin/local -p <passphrase> -s loopholes,sniffers # Run only for 2 sources
$ bin/local -p <passphrase> -s all -v # Show debug information and sync all sources
$ bin/local -p <passphrase> [-s <sources>] [-v]
```

Check logs in the `log/distribusion.log` and `STDOUT`.

TEST
====

Use minitest framework

```shell
$ rake test
```

Coverage https://67-118627778-gh.circle-artifacts.com/0/home/circleci/samples/distribusion/coverage/index.html#_AllFiles

TODO
====

- [x] Setup ruby application structure
- [x] Setup logger
- [x] Load Sentinel records
- [x] Parse Sentinel records
- [x] Load Sniffers records
- [x] Parse Sniffers records
- [x] Load Loopholes records
- [x] Parse Loopholes records
- [x] Upload Sentinel records
- [x] Upload Sniffers records
- [x] Upload Loopholes records
