---
title: Why you're probably leaking secrets in your cURL Scripts (and how to fix it)
---
There are a lot of ways to use an API token with `curl`, and most of them are insecure. 

Normally, you might see an approach like this:
```sh
API_TOKEN=$(cat $API_TOKEN_PATH)
curl --silent -X POST "$URL" \
  -F "document=@$DOC_PATH" \
  -H "Authorization: Token $API_TOKEN" \
  -H "Content-Type: multipart/form-data"
```

What's wrong with this? You'll notice we're specifying the API token for this request inside of the `$API_TOKEN` environment variable. 

When this command executes, that variable will be expanded and replaced with its real value in the command's arguments. Commands and arguments can be seen globally by other users through the `ps aux` command. There is also a risk of these commands being recorded in history or log files, where the sensitive information will also be stored in plaintext.

A more secure approach would be this:
```sh
{ echo -n 'Authorization: Token '; cat $API_TOKEN_PATH | tr -d '\n'; } | \
curl --silent -X POST "$URL" \
  -F "document=@$DOC_PATH" \
  -H @- \
  -H "Content-Type: multipart/form-data"
```

What's going on here? The main difference is the `-H @-` argument. The `@` tells `curl` that we are going to pass in the contents of the header from a file instead of as a string, while the `-` tells `curl` that the "file" we are using is stdin.

So, `-H @-` means the contents of the header will be read from stdin. This is useful because stdin/stdout are *not* visible to other users via the `ps aux` command, nor will they be recorded in histories or logs. 

This brings us to the other portion of the command:
```sh
{ echo -n 'Authorization: Token '; cat $API_TOKEN_PATH | tr -d '\n'; } | \
```

This command builds the same string that would have been passed to the `-H` argument in our insecure example, but it does so only using commands that print to stdout. Of particular note is that the actual API token value is sent directly from a file to stout with the `cat` command, leaving no opportunity for it to be leaked.

Thanks for taking the time to read this post. Hopefully this will make your applications and Jenkins pipelines a little more secure. :)

Sources: 
- https://coderwall.com/p/dsfmwa/securely-use-basic-auth-with-curl
- https://security.stackexchange.com/questions/197784/is-it-unsafe-to-use-environmental-variables-for-secret-data
- https://stackoverflow.com/questions/3830823/hiding-secret-from-command-line-parameter-on-unix#28065618