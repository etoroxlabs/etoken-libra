# eToken test scripts

Libra commit: `69cab6cc875312ae53bdc8a7f9709e39e54ed032`.

By studying the format of special comments called 'Directives' that is used in the Libra testsuite, Quantstamp developed a set of scripts to verify the functionality of eToken. Using directives, it is possible to simulate the interaction of different users by specifying the execution of multiple transactions by multiple accounts.

Using directive, it is possible to replace the hardcoded if condition in publish with an address placeholder to test the code in that would be used in production. i.e. the statement `if (true)` is replaced by `if (move(sender) == {{alice}})`. Failures are being captured explicitly, for example, the test `4_transfer_more_than_balance_should_fail` would abort at the assertion with error number `3`, this is being caught at the end of the script: with directive`// check: Aborted(3)`. The Libra testsuite would check if the failure is the same as specified, the test passes if they match.

The script and the module should be placed under libra/language/functional_tests/testsuite and have it run by the command "cargo test -p functional_tests". To run individual script, if the filename of the script is "1_owner_can_mint_and_amount_correct.mvir", then one could run the test by "cargo test -p functional_tests 1_owner_can_mint_and_amount_correct".

Unfortunately, there is currently no modularized way of testing the moveIR module. Therefore we had to clone the eToken code to every test scripts. 

For the content of the test scripts, here is a detailed description:

1. The owner publishes the modules, and can correctly mint tokens with balance correctly updated.

1. A new user can publish the module from the owner's address and query its balance.

1. The owner can transfer token to the new user.

1. It should fail when one tries to transfer more token then its own balance.

1. Regular user can transfer tokens.

1. Regular user cannot transfer tokens after being blacklisted.

1. A user (who is not the owner) that has been granted the ability to mint, can mint. 

1. Transfering tokens to a user that hasn't interacted with the token would fail.

Lastly, there is one script that is related to our findings:

1. Owner can blacklist himself and cannot transfer tokens after being blacklisted.

