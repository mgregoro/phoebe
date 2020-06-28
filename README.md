# Gemini Wiki

This server serves a wiki as a Gemini site.

It does two things:

- It's a program that you run on a computer and other people connect to it
      using their [client](https://gemini.circumlunar.space/clients.html) in
      order to read the pages on it.
- It's a wiki, which means that people can edit the pages without needing an
      account. All they need is a client that speaks both
      [Gemini](https://gemini.circumlunar.space/) and Titan, and the password.
      The default password is "hello". 😃

## How do you edit a Gemini Wiki?

You need to use a Titan-enabled client.

Known clients:

- [Gemini Write](https://alexschroeder.ch/cgit/gemini-write/) is an
      extension for the Emacs Gopher and Gemini client
      [Elpher](https://thelambdalab.xyz/elpher/)
- [Gemini & Titan for Bash](https://alexschroeder.ch/cgit/gemini-titan/about/?h=main)
      are two shell functions that allow you to download and upload files

## What is Titan?

Titan is a companion protocol to Gemini: it allows clients to upload files to
Gemini sites, if servers allow this. Gemini Wiki, you can edit the "raw" pages.
That is, at the bottom of a page you'll see a link to the "raw" page. If you
follow it, you'll see the page content as plain text. You can submit a changed
version of this text to the same URL using Titan. There is more information for
developers available [on Community Wiki](https://communitywiki.org/wiki/Titan).

## Dependencies

Perl libraries you need to install if you want to run Gemini wiki:

- [Algorithm::Diff](https://metacpan.org/pod/Algorithm::Diff)
- [File::ReadBackwards](https://metacpan.org/pod/File::ReadBackwards)
- [File::Slurper](https://metacpan.org/pod/File::Slurper)
- [Modern::Perl](https://metacpan.org/pod/Modern::Perl)
- [Net::Server](https://metacpan.org/pod/Net::Server)
- [URI::Escape](https://metacpan.org/pod/URI::Escape)

We are also going to be using `curl` and `openssl` in these installation
instructions.

On Debian:

    sudo apt install \
      libalgorithm-diff-xs-perl \
      libfile-readbackwards-perl \
      libfile-slurper-perl \
      libmodern-perl-perl \
      libnet-server-perl \
      liburi-escape-xs-perl \
      curl openssl

## Quickstart

Right now there aren't any releases. You just get the latest version from the
repository and that's it. These instructions assume that you're going to create
a new user in order to be safe.

    sudo adduser --disabled-login --disabled-password gemini
    sudo su gemini
    cd

Now you're in your home directory, `/home/gemini`. We're going to install
things right here.

    curl --output gemini-wiki.pl \
      https://alexschroeder.ch/cgit/gemini-wiki/plain/gemini-wiki.pl?h=main

Since Gemini traffic is encrypted, we need to generate a certificate and a key.
These are both stored in PEM files. To create these files, use the following:

    openssl req -new -x509 -nodes -out cert.pem -keyout key.pem

You should have three files, now: `gemini-wiki.pl`, `cert.pem`, and
`key.pem`. That's enough to get started! Start the server:

    perl gemini-wiki.pl

This starts the server in the foreground. Open a second terminal and test it:

    echo gemini://localhost \
      | openssl s_client --quiet --connect localhost:1965 2>/dev/null

You should see a Gemini page starting with the following:

    20 text/gemini; charset=UTF-8
    Welcome to the Gemini version of this wiki.

Success!! 😀 🚀🚀

Let's create a new page using the Titan protocol, from the command line:

    echo "Welcome to the wiki!" > test.txt
    echo "Please be kind." >> test.txt
    echo "titan://localhost/raw/"`date --iso-8601=date`";mime=text/plain;size="`wc --bytes < test.txt`";token=hello" \
      | cat - test.txt | openssl s_client --quiet --connect localhost:1965 2>/dev/null

You should get a nice redirect message, with an appropriate date.

    30 gemini://localhost:1965/page/2020-06-27

You can check the page, now (replacing the appropriate date):

    echo gemini://localhost:1965/page/2020-06-27 \
      | openssl s_client --quiet --connect localhost:1965 2>/dev/null

You should get back a page that starts as follows:

    20 text/gemini; charset=UTF-8
    Welcome to the wiki!
    Please be kind.

## Wiki Directory

You home directory should now also contain a wiki directory called `wiki`. In
it, you'll find a few more files:

- `page` is the directory with all the page files in it
- `index` is a file containing all the files in your `page` directory for
      quick access; if you create new files in the `page` directory, you should
      delete the `index` file – dont' worry, it will get regenerated when
      needed
- `keep` is the directory with all the old revisions of pages in it – if
      you've only made one change, then it won't exist, yet; and if you don't
      care about the older revisions, you can delete them
- `file` is the directory with all the uploaded files in it – if you
      haven't uploaded any files, then it won't exist, yet; you must explicitly
      allow MIME types for upload using the `--wiki_mime_type` option
- `meta` is the directory with all the meta data for uploaded files in it –
      there should be a file here for every file in the `file` directory; if
      you create new files in the `file` directory, you should create a
      matching file here
- `changes.log` is a file listing all the pages made to the wiki; if you
      make changes to the files in the `page` or `file` directory, they aren't
      going to be listed in this file and thus people will be confused by the
      changes you made – your call (but in all fairness, if you're collaborating
      with others you probably shouldn't do this)
- `config` probably doesn't exist, yet; it is an optional file containing
      Perl code where you can mess with the code (see _Configuration_ below)

The server uses "access tokens" to check whether people are allowed to edit
files. You could also call them passwords, if you want. They aren't associated
with a username. By default, the only password is "hello". That's why the Titan
command above contained "token=hello". 😊

## Options

The Gemini Wiki has a bunch of options, and it uses [Net::Server](https://metacpan.org/pod/Net::Server) in the
background, which has even more options. Let's try to focus on the options you
might want to use right away.

Here's an example:

    perl gemini-wiki.pl \
      --wiki_token=Elrond \
      --wiki_token=Thranduil \
      --wiki_pages=Welcome \
      --wiki_pages=About

And here's some documentation:

- `--wiki_token` is for the token that users editing pages have to provide;
      the default is "hello"; you use this option multiple times and give
      different users different passwords, if you want
- `--wiki_main_page` is the page containing your header for the main page;
      that's were you would put your ASCII art header, your welcome message, and
      so on
- `--wiki_pages` is an extra page to show in the main menu; you can use
      this option multiple times
- `--wiki_mime_type` is a MIME type to allow for uploads; text/plain is
      always allowed and doesn't need to be listed; you can also just list the
      type without a subtype, eg. `text` will allow all sorts of texts
- `--host` is the hostname to serve; the default is `localhost` – you
      probably want to pick the name of your machine, if it is reachable from
      the Internet
- `--port` is the port to use; the default is 1965
- `--wiki_dir` is the wiki data directory to use; the default is either the
      value of the `GEMINI_WIKI_DATA_DIR` environment variable, or the "./wiki"
      subdirectory
- `--cert_file` is the certificate PEM file to use; the default is
      "cert.pem"
- `--key_file` is the private key PEM file to use; the default is
      "key.pem"
- `--log_level` is the log level to use, 0 is quiet, 1 is errors, 2 is
      warnings, 3 is info, and 4 is debug; the default is 2

If you want to start the Gemini Wiki as a daemon, the following options come in
handy:

- `--setsid` makes sure the Gemini Wiki runs as a daemon in the background
- `--pid_file` is the file where the process id (pid) gets written once the
      server starts up; this is useful if you run the server in the background
      and you need to kill it
- `--log_file` is the file to write logs into; the default is to write log
      output to the standard error (stderr)
- `--user` and `--group` might come in handy if you start the Gemini Wiki
      using a different user

## Limited, read-only HTTP support

You can actually look at your wiki pages using a browser! But beware: these days
browser will refuse to connect to sites that have self-signed certificates.
You'll have to click buttons and make exceptions and all of that, or get your
certificate from Let's Encrypt or the like. Anyway, it works in theory:
`https://localhost:1965/` should work, now!

## Configuration

This section describes some hooks you can use to customize your wiki using the
`config` file.

- `@extensions` is a list of additional URLs you want the wiki to handle;
      return 1 if you handle a URL
- `@main_menu` adds more lines to the main menu, possibly links that aren't
      simply links to existing pages

The following example illustrates this:

    package Gemini::Wiki;
    use Modern::Perl;
    our (@extensions, @main_menu);
    push(@main_menu, "=> gemini://localhost/do/test Test");
    push(@extensions, \&serve_test);
    sub serve_test {
      my $self = shift;
      my $url = shift;
      my $host = $self->host();
      my $port = $self->port();
      if ($url =~ m!^gemini://$host(:$port)?/do/test$!) {
        say "20 text/plain\r";
        say "Test";
        return 1;
      }
      return;
    }
    1;
