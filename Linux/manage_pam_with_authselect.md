# How to manage PAM modules with authselect?

## Issue

- pam_faillock rules do not effect managed via authselect.
- how to enable authselect feature.

## Environment

- Red Hat Enterprise Linux 8

## Root cause

1. condition within system-auth or password-auth and the `with-faillock` feature is not enabled.

    ```bash
    $ authselect current
    Profile ID: custom/auth-policy
    Enabled features: None

    $ grep "faillock" /etc/authselect/custom/auth-policy/system-auth 
    auth        required                                     pam_faillock.so preauth silent deny=3 unlock_time=1800 {include if "with-faillock"}
    auth        required                                     pam_faillock.so authfail deny=3 unlock_time=1800       {include if "with-faillock"}
    account     required                                     pam_faillock.so                                        {include if "with-faillock"}
    ```

## Solution

1. Enable the `with-faillock` feature.

    ```bash
    authselect enable-feature with-faillock
    ```

2. Verify whether the user is locked.

    ```bash
    # faillock --user test
    test:
    When                Type  Source                                           Valid
    2021-01-28 07:58:44 RHOST ::1                                                  V
    2021-01-28 07:58:48 RHOST ::1                                                  V
    2021-01-28 07:58:51 RHOST ::1                                                  V
    ```

3. Logs when the user is locked.

    ```bash
    # grep "faillock" /var/log/secure 
    Jan 28 07:58:51 ip-172-31-10-156 sshd[5116]: pam_faillock(sshd:auth): Consecutive login failures for user test account temporarily locked
    ```
