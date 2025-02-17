---
sidebar_position: 2
---

# Sieve scripting

Sieve is a scripting language used to filter and modify email messages. It provides a flexible and powerful way to manage email messages by automatically filtering, sorting, and transforming them based on a wide range of criteria.  Rather than relying on a proprietary DSL, Stalwart SMTP uses Sieve as its default scripting language primarily because it is sufficiently powerful to handle most filtering tasks and is an established [internet standard](https://www.rfc-editor.org/rfc/rfc5228.html).

A typical Sieve script consists of one or more rules, each consisting of a test and an action. The test checks a specific attribute of the message, such as the sender's address or the subject line, while the action specifies what to do with the message if it matches the test. This manual does not cover how to write Sieve scripts but tutorials and examples can be found at [https://sieve.info](https://sieve.info).

## Scripts

Sieve scripts are specified under the `sieve.scripts.<name>` key and can be invoked directly from any of the stages of an SMTP transaction or imported from other scripts using the `include` command. In the configuration file, Sieve scripts can be either embedded as text or loaded from external files using a `file://` URL, for example:

```toml
[sieve.scripts]
script_one = '''
    require ["variables", "extlists", "reject"];

    if string :list "${env.helo_domain}" "list/blocked-domains" {
        reject "551 5.1.1 Your domain '${env.helo_domain}' has been blocklisted.";
    }
'''
script_two = "file:///opt/stalwart-smtp/etc/sieve/my-script.sieve"
```

## Context Variables

Sieve scripts executed in Stalwart SMTP have access to the following environment variables:

- `env.remote_ip`: The email client's IP address.
- `env.helo_domain`: The domain name used in the EHLO/LHLO command.
- `env.authenticated_as`: The account name used for authentication.
- `env.from`: The sender's email address specified in the `From` header.
- `env.spf`: The result of the [SPF MAIL FROM](/docs/smtp/authentication/spf) check.
- `env.spf_ehlo`: The result of the [SPF EHLO](/docs/smtp/authentication/spf) check.
- `env.dkim`: The result of the [DKIM](/docs/smtp/authentication/dkim/overview) check.
- `env.arc`: The result of the [ARC](/docs/smtp/authentication/arc) check.
- `env.dmarc`: The result of the [DMARC](/docs/smtp/authentication/dmarc) check.
- `env.iprev`: The result of the [reverse IP](/docs/smtp/authentication/iprev) check.

The envelope contents can be accessed using the [envelope test](https://www.rfc-editor.org/rfc/rfc5228.html#page-27) or through variables:

- `envelope.from`: The return path specified in the `MAIL FROM` command.
- `envelope.to`: The recipient address specified in the last `RCPT TO` command.
- `envelope.notify`: The `NOTIFY` extension parameters.
- `envelope.orcpt`: The `ORCPT` value specified with the DSN extension.
- `envelope.ret`: The `RET` value specified with the DSN extension.
- `envelope.envid`: The `ENVID` value specified with the DSN extension.
- `envelope.by_time_absolute`: The specified absolute time in the `DELIVERBY` extension.
- `envelope.by_time_relative`: The specified relative time in the `DELIVERBY` extension.
- `envelope.by_mode`: The mode specified with the `DELIVERBY` extension.
- `envelope.by_trace`: The trace settings specified with the `DELIVERBY` extension.

## Runtime Settings

Stalwart SMTP compiles all defined Sieve scripts when it starts and executes them on demand using the Sieve runtime. The runtime is configured with the following parameters which are available under the `sieve` key:

- `from-name`: Defines the default name to use for the from field in email notifications sent from a Sieve script.
- `from-addr`: Defines the default email address to use for the from field in email notifications sent from a Sieve script.
- `return-path`: Defines the default return path to use in email notifications sent from a Sieve script.
- `sign`: Lists the [DKIM](/docs/smtp/authentication/dkim/overview) signatures to add to email notifications sent from a Sieve script.
- `hostname`: Sets the local hostname to use when generating a `Message-Id` header. If no value is set, the `server.hostname` value is used instead.
- `limits.redirects`: Specifies the maximum number of `redirect` commands that a Sieve script can execute.
- `limits.out-messages`: Specifies the maximum number of outgoing email messages that a Sieve script is allowed to send.
- `limits.received-headers`: Specifies the maximum number of `Received` headers that a message can contain.
- `limits.cpu`: Specifies the maximum number of instructions that a Sieve script can execute.
- `limits.nested-includes`: Specifies the maximum number of nested includes that a script can perform.
- `limits.duplicate-expiry`: Specifies the default expiration time for the expiry Sieve test.


Example:

```toml
[sieve]
from-name = "Automated Message"
from-addr = "no-reply@foobar.org"
return-path = ""
hostname = "mx.foobar.org"
sign = ["rsa"]

[sieve.limits]
redirects = 3
out-messages = 5
received-headers = 50
cpu = 10000
nested-includes = 5
duplicate-expiry = "7d"
```

## Arithmetical and Logical Expressions

The expression extension introduces the ability to evaluate arithmetical and logical operations within Sieve scripts. This is particularly useful for performing calculations on variables or determining the flow based on specific logical conditions.
Expressions can be evaluated using the `eval` test or the `%{}` tag.

### Using the `eval` Test

Expressions can be evaluated using the `eval` test. This test evaluates the expression within the quotes and returns a boolean value based on the result.

```sieve
if test eval "expression_here" {
    # Actions to be performed if the expression evaluates to true
}
```

In the following example, if the given mathematical expression results in a value greater than `2.25`, the message is rejected as SPAM.

```sieve
if test eval "score + ((awl_score / awl_count) - score) * awl_factor > 2.25" {
    reject "Your message is SPAM.";
    stop;
}
```

### Using the `%{}` Tag

Expressions can also be evaluated using the `%{}` tag, which replaces the tag with the result of the expression.

For example, to set a variable's value based on an evaluated expression:

```sieve
set "score" "%{score + ((awl_score / awl_count) - score) * awl_factor}";
```

Or, to concatenate a string with the evaluated result:

```sieve
addheader "X-Score" "Calculated score of %{score + ((awl_score / awl_count) - score) * awl_factor}";
```

In the example above, an "X-Score" header is added to the message with the calculated score.

## Supported Operators

- **Arithmetical Operators:**
    - `+`: Addition
    - `-`: Subtraction
    - `/`: Division
    - `*`: Multiplication
    
- **Logical Operators:**
    - `&&`: Logical AND
    - `||`: Logical OR
    - `^`: Logical XOR
    - `!`: Logical NOT
    
- **Comparison Operators:**
    - `>`: Greater than
    - `<`: Less than
    - `=`: Equal to
    - `!=`: Not equal to
    - `>=`: Greater than or equal to
    - `<=`: Less than or equal to

When working with logical and comparison operators, it is crucial to ensure that the data types being compared are compatible. For instance, comparing a string to a number using an operator like `>` may not yield the expected results. Always ensure that the variables and constants in your expressions have expected and consistent types to avoid unexpected behaviors.

## Plugins extension

Stalwart SMTP adds to the Sieve language the `vnd.stalwart.plugins` extension which allows scripts to execute external commands or SQL queries using the `exec` and `query` commands (and tests). Both commands/tests expects as arguments the command or query name followed by the command arguments or query values.

### Directory queries

The `query` command is used to execute an SQL or LDAP query on the directory. It expects as the first argument the query and as second argument the query values. Additionally, the following optional parameters can be specified:

- `:use <directory name>`: Specifies the directory to use for the query. The directory name is defined in the `directory.<name>.lookup` section. If left unspecified, the first directory defined in the configuration file is used.
- `:set [var1, var2, ..., varn]`: Specifies the variables to set with the query results. The variables are set in the order they are specified in the command.

For example:

```sieve
require ["vnd.stalwart.plugins", "variables", "envelope", "reject"];

set "triplet" "${env.remote_ip}.${envelope.from}.${envelope.to}";

if not query :use \"sql\" "SELECT 1 FROM greylist WHERE addr=? LIMIT 1" ["${triplet}"] {
    query  :use \"sql\" "INSERT INTO greylist (addr) VALUES (?)" ["${triplet}"];
    reject "422 4.2.2 Greylisted, please try again in a few moments.";
}
```

Or, to set the result of the query in a variable:

```sieve
global "score";
set "awl_factor" "0.5";

query :use "sql" :set ["awl_score", "awl_count"] "SELECT score, count FROM awl WHERE sender = ? AND ip = ?" ["${env.from}", "%{env.remote_ip}"];
if eval "awl_count > 0" {
	if not query :use "sql" "UPDATE awl SET score = score + ?, count = count + 1 WHERE sender = ? AND ip = ?" ["%{score}", "${env.from}", "%{env.remote_ip}"] {
		reject "update query failed";
		stop;
	}
	set "score" "%{score + ((awl_score / awl_count) - score) * awl_factor}";
} else {
	if not query :use "sql" "INSERT INTO awl (score, count, sender, ip) VALUES (?, 1, ?, ?)" ["%{score}", "${env.from}", "%{env.remote_ip}"] {
		reject "insert query failed";
		stop;
	}
}
```

### External programs

The `exec` command is used to execute external binaries, for example:

```sieve
require "vnd.stalwart.plugins";

if exec "/opt/stalwart/bin/validate.sh" ["${env.remote_ip}", "${envelope.from}"] {
    reject "You are not allowed to send e-mails.";
}

```

## Supported extensions

The Sieve extensions supported in Stalwart SMTP are listed in the [RFCs conformed section](/docs/development/rfcs#sieve).

## Examples

The following examples show how to use Sieve scripts to implement common email filtering tasks.

### Greylisting

The following script implements a simple greylisting filter using an SQL database:

```toml
[session.rcpt]
script = "greylist"

[sieve.scripts]
greylist = '''
    require ["variables", "vnd.stalwart.execute", "envelope", "reject"];

    set "triplet" "${env.remote_ip}.${envelope.from}.${envelope.to}";

    if not query "SELECT 1 FROM greylist WHERE addr=? LIMIT 1" ["${triplet}"] {
        query "INSERT INTO greylist (addr) VALUES (?)" ["${triplet}"];
        reject "422 4.2.2 Greylisted, please try again in a few moments.";
    }
'''
```

### Domain blocklisting

The following script implements a domain blocklisting filter during the [EHLO](/docs/smtp/inbound/ehlo) phase:

```toml
[session.ehlo]
script = "is-blocked"

[sieve.scripts]
is-blocked = '''
    require ["variables", "extlists", "reject"];

    if string :list "${env.helo_domain}" "sql/blocked-domains" {
        reject "551 5.1.1 Your domain '${env.helo_domain}' has been blocklisted.";
    }
'''

[directory."sql".lookup]
blocked-domains = "SELECT 1 FROM blocked_domains WHERE domain=? LIMIT 1"

```

### Message modification

The following example modifies the incoming message by replacing the content of all HTML parts with their uppercase version and adding a custom header to each part:

```toml
[session.data]
script = "modify-message"

[sieve.scripts]
modify-message = '''
    require ["envelope", "variables", "replace", "mime", "foreverypart", "editheader", "extracttext"];

    if envelope :domain :is "to" "foobar.net" {
        set "counter" "a";
        foreverypart {
            if header :mime :contenttype "content-type" "text/html" {
                extracttext :upper "text_content";
                replace "${text_content}";
            }
            set :length "part_num" "${counter}";
            addheader :last "X-Part-Number" "${part_num}";
            set "counter" "${counter}a";
        }
    }
'''
```
