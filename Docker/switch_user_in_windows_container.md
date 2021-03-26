# How to switch user in a Windows container?

## Issue

- execute command `whoami` in a Windows ServerCore container returns `user manager\containeradministrator`
- switch `user manager\containeradministrator` to other user which has the administrator priviledge

## Solution

- UAC(User Account Control) service is not available in Windows ServerCore image, so you cannot use built-in administrator user directly. alternatively, you can create an normal user and add it to the administrator group within a Dockerfile.

    ```yaml
    FROM mcr.microsoft.com/windows/servercore:ltsc2016
    RUN NET USER sp_farm /add
    RUN NET LOCALGROUP Administrators /add sp_farm

    USER sp_farm
    SHELL ["powershell"]
    RUN whoami
    ```

- Dockerfile testing

    ```yaml
    PS C:\Users\Administrator\switchuser> docker images
    REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
    switchuser                             latest              90ecd9b64839        39 seconds ago      11.3GB
    mcr.microsoft.com/windows/servercore   ltsc2016            e9d0a8d2fc57        2 weeks ago         11.3GB

    PS C:\Users\Administrator> docker run switchuser whoami
    bd76e5fdc8e2\sp_farm

    PS C:\Users\Administrator> docker run switchuser net user

    User accounts for \\983E494312BA

    -------------------------------------------------------------------------------
    Administrator            DefaultAccount           Guest
    sp_farm
    The command completed successfully.
    ```
