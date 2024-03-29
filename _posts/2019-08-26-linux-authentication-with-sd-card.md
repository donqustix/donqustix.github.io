---
title:  "Linux authentication with an SD card"
category: linux
---

In this post, we will create a different method of user authentication in Linux using a memory card, which will eliminate the requirement of typing the password in most cases.

The idea is as follows: Whenever the user needs to log in, he inserts a special memory card that stores a token, which is then compared to one on the computer. The authentication is successful if and only if the tokens on the computer and memory card are equal. For security reasons, the tokens are regenerated in the process.

## Preparing

First, we need to write a Linux PAM module for our authentication method to work. Linux PAM seperates the authentication process into four independent management groups: auth, accout, session, password. It is sufficient for our module to implement only the first of them.

Second, we need to prepare our memory card. I will be using my MicroSD card with the memory capacity of up to 1GB, which is optimal for our purposes.

Let's wipe all data on the MicroSD card or whatever device you have with the following command:
{% highlight bash %}
dd if=/dev/zero of=/dev/mmcblk0
{% endhighlight %}

## Writing code

We are going to use the C language, since Linux PAM modules are written in C.

Having determined the programming language, let's start writing code for the SD card initialization procedure.

### The SD card part

The first function is the token generation function.
```c
int generate_token(unsigned char* data)
{
    const int urandom_fd = open("/dev/urandom", O_RDONLY);
    if (urandom_fd < 0) {
        log_error("generate_token() -> open() failed: %s", strerror(errno));
        return 1;
    }
    read(urandom_fd, data, TOKEN_SIZE);
    close(urandom_fd);
    return 0;
}
```

The second function is a function that writes the generated token to the SD card.
```c
int save_token_microsd(unsigned char* token, const char* user)
{
    const int microsd_fd = open("/dev/mmcblk0", O_WRONLY);
    if (microsd_fd < 0) {
        log_error("save_token_microsd() -> open() failed: %s", strerror(errno));
        return 1;
    }
#ifdef TOKEN_WRITE_HEADER
    const unsigned char header[HEADER_CODE_SIZE] = {
        49, 138, 84, 64, 58, 19, 175, 38, 170, 252
    };
    const size_t username_size = strlen(user) + 1;
    write(microsd_fd, header,  HEADER_CODE_SIZE);
    write(microsd_fd, user, username_size);
    lseek(microsd_fd, HEADER_USERNAME_SIZE - username_size, SEEK_CUR);
#else
    lseek(microsd_fd, HEADER_CODE_SIZE + HEADER_USERNAME_SIZE, SEEK_SET);
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
    if (!pwd) {
        log_error("getpwnam() failed: %s", strerror(errno));
        return 1;
    }
    char filepath[64];
    snprintf(filepath, 64, "%s/microsd_token", pwd->pw_dir);
    const int microsd_token_fd = open(filepath, O_WRONLY | O_CREAT);
    if (microsd_token_fd < 0) {
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
    if (save_token_microsd(user, token))
        return 1;
    if (save_token_home(user, token))
        return 1;
    return 0;
}
```

### The Linux PAM module

Our module is required to implement the following two functions from the authentication group:
```c
PAM_EXTERN int pam_sm_authenticate(pam_handle_t* pamh, int flags, int argc, const char** argv);
PAM_EXTERN int pam_sm_setcred(pam_handle_t* pamh, int flags, int argc, const char** argv);
```

In the `pam_sm_authenticate` function we obtain the username, verify the header of the SD card and the username, read the tokens from the SD card and home directory, compare the tokens, and regenerate them if no errors have occurred.
```c
PAM_EXTERN int pam_sm_authenticate(pam_handle_t* pamh, int flags, int argc, const char** argv)
{
    const char* username = NULL;
    const int pam_code = pam_get_user(pamh, &username, NULL);
    int ret_value = PAM_AUTH_ERR;
    if (pam_code != PAM_SUCCESS) {
        log_error("pam_get_user() failed: %s", pam_strerror(pamh, pam_code));
        goto cleanup;
    }
    const int microsd_fd = open("/dev/mmcblk0", O_RDONLY);
    if (microsd_fd < 0) {
        log_error("open() failed: %s", strerror(errno));
        goto cleanup;
    }
    const unsigned char header_cmp[] = {
        49, 138, 84, 64, 58, 19, 175, 38, 170, 252
    };
    struct {
        union {
            unsigned char header    [HEADER_CODE_SIZE    ];
                     char username  [HEADER_USERNAME_SIZE];
            unsigned char data      [TOKEN_SIZE          ];
        } microsd;
        unsigned char home[TOKEN_SIZE];
    } token;
    read(microsd_fd, token.microsd.header,  HEADER_CODE_SIZE);
    if (memcmp(token.microsd.header, header_cmp,  HEADER_CODE_SIZE)) {
        log_error("bad header");
        goto cleanup_microsd_fd;
    }
    const int username_size = read(microsd_fd, token.microsd.username, HEADER_USERNAME_SIZE);
    if (strcmp(token.microsd.username, username)) {
        log_error("bad user");
        goto cleanup_microsd_fd;
    }
    lseek(microsd_fd, HEADER_USERNAME_SIZE - username_size, SEEK_CUR);
    read(microsd_fd, token.microsd.data, TOKEN_SIZE);
    errno = 0;
    const struct passwd* const pwd = getpwnam(username);
    if (!pwd) {
        log_error("getpwnam() failed: %s", strerror(errno));
        goto cleanup_microsd_fd;
    }
    char buffer[64];
    snprintf(buffer, 64, "%s/microsd_token", pwd->pw_dir);
    const int token_home_fd = open(buffer, O_RDONLY);
    if (token_home_fd < 0) {
        log_error("open() - microsd_token failed: %s", strerror(errno));
        goto cleanup_microsd_fd;
    }
    read(token_home_fd, token.home, TOKEN_SIZE);
    if (memcmp(token.microsd.data, token.home, TOKEN_SIZE)) {
        log_error("bad token");
        goto cleanup_token_home_fd;
    }
    update_token(username); ret_value = PAM_SUCCESS;
cleanup_token_home_fd: close(token_home_fd);
cleanup_microsd_fd:    close(microsd_fd);
cleanup:
    return ret_value;
}
```

The `pam_sm_setcred` function has no use for us.
```c
PAM_EXTERN int pam_sm_setcred(pam_handle_t* pamh, int flags, int argc, const char** argv)
{
    return PAM_SUCCESS;
}
```

Having written the module, compile it with
```bash
gcc -fPIC -fno-stack-protector -c pam_microsd_login.c -o pam_microsd_login.o
```

and create a shared object
```bash
sudo ld -x --shared -o /lib/x86_64-linux-gnu/security/pam_microsd_login.so pam_microsd_login.o
```

## Turning the PAM module on

In order to use our module as the default authentication method, locate the following line in /etc/pam.d/common-auth (depending on the system):
```
auth    required        pam_unix.so nullok_secure
```

And change it to look something like that:
```
auth    sufficient      pam_microsd_login.so
auth    required        pam_unix.so nullok_secure
```

By default, unprivileged users are not allowed to access the `dev/mmncblk0` device. One of the ways to deal with this is to create an udev rule to set the appropriate ACL permissions.

## Get the full code on GitHub

The source code can be obtained [here]({{ site.github_url }}pam_microsd_login).
