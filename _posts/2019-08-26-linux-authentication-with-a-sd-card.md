---
mathjax: false
mermaid: false
title:  "Linux Authentication With a SD Card"
category: linux
---

In this post, we will create a different method for user authentication in Linux using a flash drive, which will eliminate the requirement of typing the password in most cases.

The idea is as follows: Everytime the user needs to log in, he inserts a special flash drive that stores a token, which is then compared to one on the computer. The authentication is successful if and only if the tokens on the computer and flash drive are equal. For security reasons, the tokens are regenerated in the process.

## Preparing

First, we need to write a Linux PAM module for our authentication method to work. Linux PAM seperates the authentication process into four independent management groups: auth, accout, session, password. It is sufficient for our module to implement only the first of them.

Second, we need to prepare our flash drive. I will be using my MicroSD card with the memory capacity of up to 1GB, which is optimal for our purposes.

Let's wipe all the data on the MicroSD card or whatever device you have with the following command:
{% highlight bash %}
dd if=/dev/zero of=/dev/mmcblk0
{% endhighlight %}

## Writing Code

We are going to use the C language, since Linux PAM modules are written in C.

Having the programming language determined, let's start writing code for the SD card initialization procedure.

### The SD Card Part

The first function is the token generator function.
```c
int generate_token(unsigned char* data)
{
    const int urandom_fd = open("/dev/urandom", O_RDONLY);
    if (urandom_fd < 0)
    {
        log_error("generate_token() -> open() failed: %s", strerror(errno));
        return 1;
    }
    read(urandom_fd, data, TOKEN_SIZE);
    close(urandom_fd);
    return 0;
}
```

The second function is a function for writing the generated token to the SD card.
```c
int save_token_microsd(unsigned char* token)
{
    const int microsd_fd = open("/dev/mmcblk0", O_WRONLY);
    if (microsd_fd < 0)
    {
        log_error("save_token_microsd() -> open() failed: %s", strerror(errno));
        return 1;
    }
#ifdef TOKEN_WRITE_HEADER
    const unsigned char header[] = {
        49, 138, 84, 64, 58, 19, 175, 38, 170, 252
    };
    write(microsd_fd, header, sizeof header);
#else
    lseek(microsd_fd, 10, SEEK_SET);
#endif
    write(microsd_fd, token, TOKEN_SIZE);
    close(microsd_fd);
    return 0;
}
```

Note the header array in the code above; it is read from the SD card and compared by the PAM module in order to proceed the authentication.

We also need a function that will be writing the generated token to the home directory.
```c
int save_token_home(const char* user, unsigned char* token)
{
    errno = 0;
    const struct passwd* const pwd = getpwnam(user);
    if (!pwd)
    {
        log_error("getpwnam() failed: %s", strerror(errno));
        return 1;
    }

    char filepath[64];
    snprintf(filepath, 64, "%s/microsd_token", pwd->pw_dir);

    const int microsd_token_fd = open(filepath, O_WRONLY | O_CREAT);
    if (microsd_token_fd < 0)
    {
        log_error("save_token_home() -> open() failed: %s", strerror(errno));
        return 1;
    }
    write(microsd_token_fd, token, TOKEN_SIZE);
    close(microsd_token_fd);
    return 0;
}
```

Having all the necessary functions, we can now initialize the SD card with a token.
```c
int update_token(const char* user)
{
    unsigned char token[TOKEN_SIZE];
    if (generate_token(token))
        return 1;
    if (save_token_microsd(token))
        return 1;
    if (save_token_home(user, token))
        return 1;
    return 0;
}
```

### The Linux PAM Module

Our module is required to implement the following two functions from the authentication group:
```c
PAM_EXTERN int pam_sm_authenticate(pam_handle_t* pamh, int flags, int argc, const char** argv);
PAM_EXTERN int pam_sm_setcred(pam_handle_t* pamh, int flags, int argc, const char** argv);

```

In the `pam_sm_authenticate` function we obtain the username, check the header of the SD card, read the tokens from the SD card and home directory, compare the tokens, and regenerate them if no errors have occurred.
```c
PAM_EXTERN int pam_sm_authenticate(pam_handle_t* pamh, int flags, int argc, const char** argv)
{
    const char* username = NULL;
    const int pam_code = pam_get_user(pamh, &username, NULL);
    if (pam_code != PAM_SUCCESS)
    {
        log_error("pam_get_user() failed: %s", pam_strerror(pamh, pam_code));
        return PAM_AUTH_ERR;
    }
    const int microsd_fd = open("/dev/mmcblk0", O_RDONLY);
    if (microsd_fd < 0)
    {
        log_error("open() failed: %s", strerror(errno));
        return PAM_AUTH_ERR;
    }
    const unsigned char header_cmp[] = {
        49, 138, 84, 64, 58, 19, 175, 38, 170, 252
    };
    struct
    {
        union
        {
            unsigned char header[10];
            unsigned char data[TOKEN_SIZE];
        };
        bool good;
    } token_microsd;
      token_microsd.good = false;
    read(microsd_fd, token_microsd.header, sizeof token_microsd.header);
    if (memcmp(token_microsd.header, header_cmp, 10))
        log_error("bad header");
    else
    {
        read(microsd_fd, token_microsd.data, sizeof token_microsd.data);
        token_microsd.good = true;
    }
    close(microsd_fd);
    if (token_microsd.good)
    {
        errno = 0;
        const struct passwd* const pwd = getpwnam(username);
        if (!pwd)
            log_error("getpwnam() failed: %s", strerror(errno));
        else
        {
            char buffer[64];
            snprintf(buffer, 64, "%s/microsd_token", pwd->pw_dir);
            const int token_home_fd = open(buffer, O_RDONLY);
            if (token_home_fd < 0)
                log_error("open() - microsd_token failed: %s", strerror(errno));
            else
            {
                unsigned char token_home[TOKEN_SIZE];
                read(token_home_fd, token_home, TOKEN_SIZE);
                close(token_home_fd);
                if (!memcmp(token_microsd.data, token_home, TOKEN_SIZE))
                {
                    update_token(username);
                    return PAM_SUCCESS;
                }
                log_error("bad token");
            }
        }
    }
    return PAM_AUTH_ERR;
}
```

The `pam_sm_setcred` function has no use for us.
```c
PAM_EXTERN int pam_sm_setcred(pam_handle_t* pamh, int flags, int argc, const char** argv)
{
    return PAM_SUCCESS;
}
```

Having the module written, compile it with
```bash
gcc -fPIC -fno-stack-protector -c pam_microsd_login.c -o pam_microsd_login.o
```

and create the shared object
```bash
sudo ld -x --shared -o /lib/x86_64-linux-gnu/security/pam_microsd_login.so pam_microsd_login.o
```

## Turning the PAM Module On

In order to use our module as the default authentication method, locate the following line in /etc/pam.d/common-auth (depending on the system):
```
auth    required        pam_unix.so nullok_secure
```

And change it to look something like that:
```
auth    sufficient      pam_microsd_login.so
auth    required        pam_unix.so nullok_secure
```

By default, unprivileged users are not allowed to access the `dev/mmncblk0` device. One of the ways to deal with it is to create an udev rule to set the appropriate ACL permissions.

## Get the Full Code on GitHub

The source code can be obtained [here](https://github.com/donqustix/pam_microsd_login).
