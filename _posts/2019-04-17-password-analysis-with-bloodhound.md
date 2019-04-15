---
layout: post
title: Case Study - Password Analysis with BloodHound
subtitle: Sniffing out password reuse
image: /img/misc/BloodHound.png
tags: [python, bloodhound]
---

BloodHound is a fantastic tool for analyzing Active Directory, but that's really just the beginning. The BloodHound database is a resource that enables users to execute all sorts of queries -- the only limit is your creativity. In this case study, the SpecterOps team used BloodHound to assist with an analysis of password hashes from two different domains.

# The Setup

First, collect two lists of  typical NTLM hashes, formatted like this:

`DOMAIN\USERNAME:::32-CHAR-NTLM-HASH:::`

For this case study, SpecterOps collected two lists of NTLM hashes from two different domains, but the analysis could be done with one big list for analyzing how many accounts in the same domain share a password.

It would be simple to compare these lists using basic \*nix commands to split the strings and check if the hash in list one appeared in list two; however, that doesn't account for disabled accounts. Some organizations will set a default password for newly created or disabled accounts which will make any script looking for password reuse go berserk.

# Enter BloodHound

BloodHound enhances our analysis by keeping track of several useful pieces of information for every account: what level of privilege the user has, what AD security groups it effectively belongs to, and whether the account is even enabled in the first place. If the script detects password reuse, and checks in with BloodHound before logging it, there is an opportunity to make one very smart analysis script. This can be accomplished using the Python 3.7 `neo4j` library and Bolt driver. The provided snippets will assume you have a BloodHound database and Neo4j is running.

Note that for this to be as accurate as possible the hash and BloodHound collections should be performed at the same time. Otherwise, password resets may occur in-between collections and the `pwdlastset` values might be incorrect. It is a small thing (the hashes are much more important) but could cause some confusion.

## Connecting to BloodHound

The following function will set up a connection to a local Neo4j graph database using the provided username and password and then return the Bolt driver object. It's really just one line of code but it's always a good idea to include some error handling and user feedback.

```python
from neo4j import GraphDatabase

def setup_database_conn():
    """Function to setup the database connection to the Neo4j project containing the BloodHound data."""
    try:
        database_uri = 'bolt://localhost:7687'
        database_user = 'neo4j'
        database_pass = 'bloodhound'
        print("[!] Attempting to connect to your Neo4j project using {}:{} @ {}.".format(database_user, database_pass, database_uri))
        neo4j_driver = GraphDatabase.driver(database_uri, auth=(database_user, database_pass))
        print("[+] Success!")
        return neo4j_driver
    except:
        neo4j_driver = None
        print("[X] Database connection failed!")
        exit()
```

Now the `setup_database_conn()` function can be called at the start of the script to prepare for executing queries.

## Executing Queries

The next function executes the query. In this case the query is a simple `MATCH` query that filters out any users BloodHound has marked as disabled. There are a couple of things that are different from what is normally done for a Cypher query. Take a look and see if you notice anything unusual.

```python
def execute_query(driver, user, enabled=True):
    """Execute the provided query using the current Neo4j database connection."""
    if enabled:
        query = """
        MATCH (u:User)
        WHERE u.name =~ UPPER('%s@.*') AND (u.enabled = True)
        RETURN u.enabled, u.pwdlastset, u.domain
        """ % user  
    else:
        query = """
        MATCH (u:User)
        WHERE u.name =~ UPPER('%s@.*')
        RETURN u.enabled, u.pwdlastset, u.domain
        """ % user  
    with driver.session() as session:
        results = session.run(query)
    return results
```

First, the query uses the `WHERE` clause to filter on the user's `name` label. This could be done in the `MATCH` using `MATCH (u:User {name:'FOO@BAR.COM'})` but there are two things to be mindful of when constructing the query. The first thing is BloodHound stores all user objects as `USERNAME@DOMAIN.COM`. A search for just a username will return nothing so the `WHERE` looks for a `name` that is like (`=~`) the username + `@.*`. The `.*` is Java’s wildcard character and the `@` ensures the query doesn't accidentally match multiple usernames.

The other thing is usernames in the dumps may be a mix of upper and lowercase. BloodHound stores all names in uppercase and Cypher is case sensitive. This is easy to fix by wrapping the `name` the query looks for in the `UPPER()` function. This way any name provided to the query will be converted to all uppercase.

An example might help make sense of these considerations.

Let's say there are two usernames in the dump: CHRISM and CHRISM_da

A query for `MATCH (u:User) WHERE u.name =~ 'CHRISM.*' RETURN u` would return _both_ usernames because `CHRISM.*` matches `CHRISM_da`. This is fixed by adding the `@` to restrict the wildcard to only the domain part of the username. `MATCH (u:User) WHERE u.name =~ 'CHRISM@.*' RETURN u` will only return `CHRISM@DOMAIN.COM` from BloodHound.

The other username has lowercase characters. The above query won't return any results for `CHRISM_da` because it doesn't match the version with all uppercase characters in BloodHound. To account for any instances of lowercase characters the query can include `UPPER()` without causing any harm to usernames that are already uppercase.

Finally, the function returns the query’s results which will contain the account’s pwdlastset timestamp, domain, and enabled status (a boolean value). The Bolt driver will return a BoltStatementResult object that must be enumerated, even if there is just one match. The enumeration could be done inside this function, but that would limit the results to just the first match. Your dataset might have identical usernames in different domains.

It is also useful to be able to check if there query returned any matches. This is accomplished using the Bolt driver's `peek()` function to check if there are any results.

```python
def check_records(results):
    """Checks if the Cypher results are empty or not."""
    if results.peek():
        return True
    else:
        return False
```

If `peek()` finds anything the function will return `True`.

## Parsing the Hashes

The Neo4j and BloodHound portion is ready to go so now the script needs to parse hashes. The following snippet can be used multiple times to parse as many files as are needed. This snippet opens the file for reading and creates a dictionary to hold the hashes. Then it loops through the lines of the file and filters out machine accounts (account names ending with `$`) and any accounts with the NTLM hash `31d6cfe0d16ae931b73c59d7e0c089c0` (empty / no password).

```python
def process_hashes(hash_file):
    """Process the hashes in the provided file and return a dictionary."""
    # Create hashes of the hashes, lol
    with open(hash_file, 'r') as hash_dump:
        hashes = {}
        for line in hash_dump:
            # Ignore machine accounts
            if not '$' in line:
                # Separate DOMAIN\USER from NTLM and USER from DOMAIN
                array = line.split(':::')
                username = array[0].split('\\')[1]
                # Ignore the empty/no password NTLM hash
                if not array[1] == '31d6cfe0d16ae931b73c59d7e0c089c0':
                    hashes[username] = array[1].strip()
    return hashes
```

The same sort of thing can be done with a Hashcat potfile. Then the dictionaries can be compared later to check if the hash has been cracked.

```python
def process_potfile(hashcat_potfile):
    """Process the provided Hashcat potfile to return a dictionary of hash values and plaintext values."""
    with open(hashcat_potfile, 'r') as potfile:
        potfile_hashes = {}
        for line in potfile:
            # This doesn't account for potfile entries for NTLMv2, etc.
            array = line.split(':')
            if len(array) > 2:
                pass
            else:
                pass_hash = array[0]
                plaintext = array[1].strip()
                potfile_hashes[pass_hash] = plaintext
    return potfile_hashes
```

There is one last function that may be helpful. A final report probably should not have the full NTLM hash so the script can sanitize the hashes for reporting purposes. This function takes a 32-character NTLM hash and replaces all but the first and last four characters with asterisks. This is adapted from Carrie Roberts' DPAT.py script

```python
def sanitize(string):
    """Sanitize the provided string by replacing chunks with asterisks."""
    sanitized_string = string
    length = len(string)
    if length == 32:
        sanitized_string = string[0:4] + "*"*(length-8) + string[length-5:length-1]
    elif length > 2:
        sanitized_string = string[0] + "*"*(length-2) + string[length-1]
    return sanitized_string
```

## The Final Loop

All that is left to do is compare the dictionaries of usernames and NTLM hash values. This snippet checks if a hash from the first list exists in the second list. If there is a match it loops through the second list looking for all matching hashes. This is the first place this snippet can be customized. If you only desire to know if a user in Domain A has the same password as any account in Domain B the comparisons can stop here. The loop through the second list catches instances where the user in Domain A has the same password as two or more users in Domain B.

The final output is a dictionary that can parsed and used as JSON for additional reporting.

```python
def compare_dumps(first_hashdump, second_hashdump):
    """Compare the two password dumps and return a dictionary of the results. JSON output:
    {
    "accounts": {
        "CHRISM": {
            "enabled": true,
            "pwdlastset": "2019-04-14 22:53:08",
            "domain": "DOMAIN.COM"
            },
            "matching_accounts": {}
                "CHRISM_DA": {
                    "enabled": true,
                    "pwdlastset": "2019-04-13 13:37:12",
                    "domain": "SECURE.DOMAIN.COM"
                }
            },
            "hash": {
                "value": "32-CHAR NTLM HASH VALUE",
                "sanitized_value": "f538************************bb75",
                "matched_value": "32-CHAR NTLM HASH VALUE",
                "sanitized_matched_value": "f538************************bb75",
                "cracked": false
            }
        },

        ...

    },
    "results": {
        "total": TOTAL MATCHES WITH ENABLED ACCOUNTS IN SECOND DUMP,
        "disabled_matches": TOTAL MATCHES WITH DISABLED ACCOUNTS IN SECOND DUMP,
        "unique_accounts": TOTAL UNIQUE MATCHES FROM FIRST DUMP
    }
    """
    # Setup dictionaries and counters for matches
    shared_passwords = {}
    shared_passwords['results'] = {}
    shared_passwords['accounts'] = {}
    shared_password_count = 0
    disabled_shared_password_count = 0
    # Connect to the Neo4j database
    neo4j_driver = setup_database_conn()
    # Process the hash dumps and potfile
    first_hashes = process_hashes(first_hashdump)
    second_hashes = process_hashes(second_hashdump)
    potfile_hashes = process_potfile('hashcat.potfile')
    # Create a progress bar because this can take a while
    with click.progressbar(length=len(first_hashes), label='Sniffing through hashes') as bar:
        # Loop over every username and hash in the first list
        for first_username, first_ntlm_hash in first_hashes.items():
            # Proceed only if the NTLM hash appears as a value in the second dump
            if first_ntlm_hash in second_hashes.values():
                # Loop over every username nad hash in the second list
                for second_username, second_ntlm_hash in second_hashes.items():
                    # Proceed if the first NTLM hash matches this NTLM hash
                    if first_ntlm_hash == second_ntlm_hash:
                        # Get the user data from BloodHound
                        first_results = execute_query(neo4j_driver, first_username)
                        if check_records(first_results):
                            # Enumerate over the results (there _could_ be more than one)
                            for first_record in first_results:
                                first_enabled = first_record[0]
                                first_pwdlastset = datetime.utcfromtimestamp(first_record[1]).strftime('%Y-%m-%d %H:%M:%S')
                                first_domain = first_record[2]
                                # Only continue if this first account is enabled (it should be based on the query)
                                if first_enabled:
                                    # Record the account details in the dictionary
                                    shared_passwords['accounts'][first_username] = {}
                                    shared_passwords['accounts'][first_username]['username'] = first_username.upper() + '@' + first_domain
                                    shared_passwords['accounts'][first_username]['enabled'] = first_enabled
                                    shared_passwords['accounts'][first_username]['pwdlastset'] = first_pwdlastset
                                    shared_passwords['accounts'][first_username]['domain'] = first_domain
                                    shared_passwords['accounts'][first_username]['matching_accounts'] = {}
                                    # Check if the hash was cracked
                                    cracked = False
                                    if first_ntlm_hash in potfile_hashes.keys():
                                        cracked = True
                                    # Add details about the hash to the entry for this account
                                    shared_passwords['accounts'][first_username]['hash'] = {}
                                    shared_passwords['accounts'][first_username]['hash']['value'] = first_ntlm_hash
                                    shared_passwords['accounts'][first_username]['hash']['sanitized_value'] = sanitize(first_ntlm_hash)
                                    shared_passwords['accounts'][first_username]['hash']['matched_value'] =second_ntlm_hash
                                    shared_passwords['accounts'][first_username]['hash']['sanitized_matched_value'] = sanitize(second_ntlm_hash)
                                    shared_passwords['accounts'][first_username]['hash']['cracked'] = cracked
                                    # Record a plaintext value if the hash was cracked
                                    if cracked:
                                        shared_passwords['accounts'][first_username]['hash']['plaintext'] = potfile_hashes[first_ntlm_hash]
                                    # Collect the second account from BloodHound
                                    second_results = execute_query(neo4j_driver, second_username)
                                    # This time proceed if there are results or record the result as a disabled account match
                                    if check_records(second_results):
                                        for second_record in second_results:
                                            second_enabled = second_record[0]
                                            second_pwdlastset = datetime.utcfromtimestamp(second_record[1]).strftime('%Y-%m-%d %H:%M:%S')
                                            second_domain = second_record[2]
                                            if second_enabled:
                                                # Track accounts in list two that share a password with a list one account
                                                shared_password_count += 1
                                                # Add the results to the 'matching_account' key
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username] = {}
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username]['username'] = second_username.upper() + '@' + second_domain
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username]['enabled'] = second_enabled
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username]['pwdlastset'] = second_pwdlastset
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username]['domain'] = second_domain
                                    else:
                                        # Same as above but for disabled accounts
                                        second_results = execute_query(neo4j_driver, second_username, enabled=False)
                                        if check_records(second_results):
                                            disabled_shared_password_count += 1
                                            for second_record in second_results:
                                                second_enabled = second_record[0]
                                                second_pwdlastset = datetime.utcfromtimestamp(second_record[1]).strftime('%Y-%m-%d %H:%M:%S')
                                                second_domain = second_record[2]
                                                # Add the results to the 'matching_account' key
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username] = {}
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username]['username'] = second_username.upper() + '@' + second_domain
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username]['enabled'] = second_enabled
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username]['pwdlastset'] = second_pwdlastset
                                                shared_passwords['accounts'][first_username]['matching_accounts'][second_username]['domain'] = second_domain
            # Manually update the progress by 1 for each account loop
            bar.update(1)
    # Record the final numeric results in the 'results' key in the dictionary
    shared_passwords['results']['total'] = shared_password_count
    shared_passwords['results']['disabled_matches'] = disabled_shared_password_count
    shared_passwords['results']['unique_accounts'] = len(shared_passwords['accounts'])
```

The important piece is the `execute_query()` function calls. These check in with BloodHound to determine if the provided username is enabled. This way disabled accounts can be ignored. Again, the script can be customized based on your needs. Running this function just once ensures the account in Domain A is enabled but does not check if the account in Domain B is enabled. If you only want to know if two enabled accounts share a password then the query can be run for both accounts. If you would like to know if an engineer in Domain A is reusing their password for a bunch of test accounts that may or may not be enabled in Domain B then the query can be run just the one time.

Furthermore, the example query only uses a few BloodHound attributes. The query can be updated to return more information about the user, such as their display name and email address. The script could also include additional queries, like checking if accounts are in a high value group (e.g. Domain Admins).

# Wrap Up

If you needed more of a reason to leverage BloodHound for defense maybe these automation possibilities will offer some encouragement! A few Cypher queries or lines of Python code can answer questions about Active Directory that you cannot easily obtain from AD itself. This case study shows how the data can also be leveraged for other projects that don’t directly involve AD.

Additionally, this Python approach offers the benefit of allowing you to run more advanced queries. There are some queries that simply will not complete when run via Neo4j’s web console but will complete when run using a bolt driver and your favorite scripting language.

Andy Robbins (@_wald0) has a pair of articles from 2018 that offer a good primer on using BloodHound data to perform deeper analysis of Active Directory resilience and generating statistics:

* [Introducing Adversary Resilience Methodology: Part 1](https://posts.specterops.io/introducing-the-adversary-resilience-methodology-part-one-e38e06ffd604)
* [Introducing Adversary Resilience Methodology: Part 2](https://posts.specterops.io/introducing-the-adversary-resilience-methodology-part-two-279a1ed7863d)

Andy ([@_wald0](https://twitter.com/_wald0)) and Rohan Vazarkar ([@cptjesus](https://twitter.com/cptjesus)) also recently presented an excellent talk on this topic at TROOPERSCon 2019:

<iframe width="560" height="315" src="https://www.youtube.com/embed/0r8FzbOg2YU" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Happy hunting! Aroooo!
