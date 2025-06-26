# filter-dnsblscore

## Description
This filter performs DNSBL lookups against a list of domains and allows
OpenSMTPD to either block or slow down a session based on the number of
blocklists the source IP address appears on.

Each IP address is assigned a score representing the number of blocklists the
IP address is found on; the higher the score, the more likely a message is to
be spam.

This filter is a fork of
[filter-senderscore](https://github.com/poolpOrg/filter-senderscore).

## Features
The filter currently supports:

- blocking hosts with reputation below a certain value
- adding an `X-DNSBL-Score` header with the score of the source IP address
- adding an `X-Spam` header to hosts with reputation below a certain value
- applying a time penalty proportional to the IP reputation
- allowlisting IP addresses or subnets


## Dependencies
The filter is written in Golang and doesn't have any dependencies beyond the standard library.

It requires OpenSMTPD 6.6.0 or higher.

## How to install
Clone the repository, build and install the filter:
```
$ cd filter-dnsblscore/
$ go build
$ doas install -m 0555 filter-dnsblscore /usr/local/bin/filter-dnsblscore
```

On Linux, use sudo(8) instead of doas(1).

## How to configure
The filter itself requires no configuration.

It must be declared in smtpd.conf and attached to a listener:
```
filter "dnsblscore" proc-exec "/usr/local/bin/filter-dnsblscore -junkAbove 0 -blockAbove 1 -slowFactor 1000 b.barracudacentral.org bl.spamcop.net"

listen on all filter "dnsblscore"
```

`-blockAbove` will display an error banner for sessions with reputation score strictly above value then disconnect.

`-blockPhase` will determine at which phase `-blockAbove` will be triggered, defaults to `connect`, valid choices are `connect`, `helo`, `ehlo`, `starttls`, `auth`, `mail-from`, `rcpt-to` and `quit`. Note that `quit` will result in a message at the end of a session and may only be used to warn sender that reputation is degrading as it will not prevent transactions from succeeding.

`-junkAbove` will prepend the 'X-Spam: yes' header to messages.

`-slowFactor` will delay all answers to a reputation-related percentage of its value in milliseconds. The formula is `delay * score / domains` where `delay` is the argument to the `-slowFactor` parameter, `score` is the reputation score, and `domains` is the number of blocklist domains. By default, connections are never delayed.

`-scoreHeader` will add an X-DNSBL-Score header with reputation value if known.

`-allowlist <file>` can be used to specify a file containing a list of IP addresses and subnets in CIDR notation to allowlist, one per line. IP addresses matching any entry in that list automatically receive a score of 0.
