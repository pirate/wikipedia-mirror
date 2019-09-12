<div align="center" style="text-align:center">

<h1>How to self-host a mirror of Wikipedia.org:<br/>with Nginx, Kimix, or MediaWiki/XOWA + Docker</h1>
<i>Originally published 2019-09-08 on <a href="https://docs.sweeting.me/s/blog">docs.sweeting.me</a>.<br/>The pretty <a href="https://docs.sweeting.me/s/self-host-a-wikipedia-mirror">HTML version is here</a> and the <a href="https://github.com/pirate/wikipedia-mirror">source for this guide is on Github</a>.</i><br/><br/>
A summary of how to set up a full Wikipedia.org mirror using three different approaches.
<hr/>
<img src="https://chrischapman.co/images/kiwix/home-page-internal.png" width="500px"/>
</div>

# Intro

> **Did you know that Wikipedia.org just runs a mostly-traditional LAMP stack on [~350 servers](https://meta.wikimedia.org/wiki/Wikimedia_servers)**? (as of 2019)

**Unfortunately, Wikipedia attracts lots of hate from people and nation-states who object to certain articles or want to hide information from the public eye.**

Wikipedia's infrastructure (2 racks the USA, 1 in Holland, and 1 in Singapore, + CDNs) [cant always stand up to large DDoS attacks](https://wikimediafoundation.org/news/2019/09/07/malicious-attack-on-wikipedia-what-we-know-and-what-were-doing/), but thankfully they provide regular database dumps and static HTML archives to the public, and have permissive licensing that allows for rehosting with modification (even for profit!).

Growing up in China [behind the GFC I often experienced Wikipedia unavailability](https://www.cnet.com/news/the-great-firewall-of-china-blocks-off-wikipedia/), and in light of the [recent DDoS](https://wikimediafoundation.org/news/2019/09/07/malicious-attack-on-wikipedia-what-we-know-and-what-were-doing/) I decided to make a guide for people to help demystify the process of running a mirror. I'm also a big advocate for free access to information, and I'm the maintainer of a major internet archiving project called [ArchiveBox](https://archivebox.io) (a self-hosted internet archiver powered by headless Chromium).

**This aim of this guide is to encourage people to use these publicly available dumps to host Wikipedia mirrors, so that malicious actors don't succeed in limiting public access to one of the *world's best sources of information*.**

## Getting Started

Wikipedia.org itself is powered by a PHP backend called [WikiMedia](https://en.wikipedia.org/wiki/MediaWiki), using MariaDB for data storage, Varnish and Memcached for request and query caching, and ElasticSearch for full-text search. Production Wikipedia.org also runs a number of extra plugins and modules on top of MediaWiki.

**🖥 There are several ways to host your own mirror of Wikipedia (with varying complexity):**

1. [**Run a caching proxy in front of Wikipedia.org**](#) (disk used on-demand for cache, low CPU use)
2. [**Serve the static HTML ZIM archive with Kiwix**](#) (~80GB for compressed archive, low CPU use)
3. [**Run a full MediaWiki server**](#) (hardest to set up, ~600GB for XML & database, high CPU use)

**💅Don't expect it to look perfect on the first try**

Setting up a Wikipidea mirror involves a complex dance between software, data, and sysadmin, so beginners are encouraged to start with the static html archive or proxy and before attempting to run a full MediaWiki Server. Users should expect their mirrors to be able to serve text articles without images, but should not expect it to look like Wikipedia.org on the first try, or the second, or the third...

**✅ Choosing an approach**

Each method in this guide has its pros and cons. A caching proxy is the most lightweight option, but if the upstream servers go down and a request comes in that hasn't been seen before and cached it will 404, so it's not a fully redundant mirror. The static ZIM mirror is lightweight to download and host (and requests are easy to cache) but it has no interactivity, full-text search capability, or Wikipedia-style category index pages. MediaWiki/XOWA are the most complex, but they can provide a full working Wikipedia mirror complete with history revisions, users, talk pages, search, and more. 

Running a full MediaWiki server is by far the hardest method to set up. Expect it to take multiple days/weeks depending on available system resources, and expect it to look fairly broken since the production Wikipedia.org team run many tweaks and plugins that take extra work to set up locally.

For more info, see the [Wikipedia.org index of all dump types available, with descriptions](https://dumps.wikimedia.org/).


## Responsible Rehosting Warning

⚠️ Be aware that running a publicly-accessible mirror of Wikipedia.org with any kind of framing / content modifications / ads is *strongly discouraged*. Framing mirrors / proxy mirrors are still a good option for private use, but you need to take additional steps to mirror responsibly if you're setting up a proxy for public use (e.g. robots:noindex, takedown contact info, blocking unlicensed images, etc.).

> <span style="font-size:14px">Some mirrors load a page from the Wikimedia servers directly every time someone requests a page from them. They alter the text in some way, such as framing it with ads, then send it on to the reader. **This is called remote loading, and it is an unacceptable use of Wikimedia server resources.** Even remote loading websites with little legitimate traffic can generate significant load on our servers, due to search engine web crawlers.
*https://en.wikipedia.org/wiki/Wikipedia:Mirrors_and_forks#Remote_loading*</span>


Luckily, regardless of how you choose to rehost Wikipedia ***text***, you are not breaking any terms and conditions or violating copyright law as long as you don't remove their copyright statements (however, note the article images and videos on Wikimedia.org may not be licensed for re-use).

> <span style="font-size: 14px">Every contribution to the English Wikipedia has been licensed for re-use, including commercial, for-profit websites. Republication is not necessarily a breach of copyright, so long as the appropriate licenses are complied with.
*https://en.wikipedia.org/wiki/Wikipedia:Mirrors_and_forks#Things_you_need_to_know*
</span>

---

# [Table of Contents](https://docs.sweeting.me/s/self-host-a-wikipedia-mirror#TOC)

[TOC]

See the [HTML version](https://docs.sweeting.me/s/self-host-a-wikipedia-mirror#TOC) of this guide for the best browsing experience. See [pirate/wikipedia-mirror](https://github.com/pirate/wikipedia-mirror) on Github for example config source, docker-compose files, binaries, folder structure, and more.

---

# Tutorial

---

## Prerequisites

1. **Provision a server to act as your Wikipedia mirror**

   You can use a cheap VPS provider like DigitalOcean, Vultr, Hertzner, etc. For the static ZIM archive and MediaWiki server methods you will need significant disk space, so a home server with a cheap external HD may be a better option.
   
   *The setup examples below are based on Ubuntu 19.04* running on a home server, however they should work across many other OS's with minimal tweaking (e.g. FreeBSD, macOS, Arch, etc.).

2. **Purchase a new domain or create a subdomain to host your mirror**

   You can use Google Domains, NameCheap, GoDaddy, etc. any registrar will work.

   *In the setup examples below, replace `wiki.example.com` with the domain you chose.*

3. **Point the DNS records for the domain to your mirror server**

   Configure these records via your DNS provider (e.g. NameCheap, DigitalOcean, CloudFlare, etc.):

   - `wiki.example.com` `A` -> `your server's public ip` (the root domain)
   - `en.wiki.example.com` `CNAME` -> `wiki.example.com` (the wiki domain)
   - `upload.wiki.example.com` `CNAME` ->  `wiki.example.com` (the uploads/media domain)

4. **Create a directory to store the project, and a dotenv file for your config options**

    Not all of these values are needed for all the methods, but it's easier to just define all of them in one place and remove things later that turn out to be unneeded.

    ```bash
    mkdir -p /opt/wiki                  # change PROJECT_DIR below to match
    nano /opt/wiki/.env
    ```
    Create the `.env` config file in [`dotenv`](https://docs.docker.com/compose/env-file/)/`bash` syntax with the contents below.
    *Make sure to replace the example values like `wiki.example.com` with your own.*
    ```bash
    PROJECT_DIR="/opt/wiki"                   # folder for all project state
    CONFIG_DIR="$PROJECT_DIR/etc/nginx"
    CACHE_DIR="$PROJECT_DIR/data/cache"
    CERTS_DIR="$PROJECT_DIR/data/certs"
    LOGS_DIR="$PROJECT_DIR/data/logs"

    LANG="en"                                 # Wikipedia language to mirror
    LISTEN_PORT_HTTP="80"                     # public-facing HTTP port to bind
    LISTEN_PORT_HTTPS="443"                   # public-facing HTTPS port to bind
    LISTEN_HOST="wiki.example.com"            # root domain to listen on
    LISTEN_WIKI="$LANG.$LISTEN_HOST"          # wiki domain to listen on
    LISTEN_MEDIA="upload.$LISTEN_HOST"        # uploads domain to listen on

    UPSTREAM_HOST="wikipedia.org"             # main upstream domain
    UPSTREAM_WIKI="$LANG.$UPSTREAM_HOST"      # upstream domain for wiki
    UPSTREAM_MEDIA="upload.wikimedia.org"     # upstream domain for uploads

    # Only needed if using an nginx reverse proxy:
    SSL_CRT="$CERTS_DIR/$LISTEN_HOST.crt"
    SSL_KEY="$CERTS_DIR/$LISTEN_HOST.key"
    SSL_DH="$CERTS_DIR/$LISTEN_HOST.dh"

    CACHE_SIZE="100G"                         # or "500GB", "1GB", "200MB", etc.
    CACHE_REQUESTS="GET HEAD POST"            # or "GET HEAD", "any", etc.
    CACHE_RESPONSES="200 206 302"             # or "200 302 404", "any", etc.
    CACHE_DURATION="max"                      # or "1d", "30m", "12h", etc.

    ACCESS_LOG="'$LOGS_DIR/nginx.out' trace"  # or "off", etc.
    ERROR_LOG="'$LOGS_DIR/nginx.err' warn"    # or "off", etc.
    ```
    
    *<span style="color:orange">The setup steps below depend on this file existing and the config values being correct,</span>
    so make sure you create it and replace all example values with your own before proceeding!*

---

## Choosing a Wikipedia archive dump

- https://en.wikipedia.org/wiki/MediaWiki
- https://www.mediawiki.org/wiki/MediaWiki
- https://www.mediawiki.org/wiki/Download
- https://www.wikidata.org/wiki/Wikidata:Database_download
- https://dumps.wikimedia.org/backup-index.html

### ZIM Static HTML Dump

Wikipedia HTML dumps are provided in a highly-compressed web-archiving format called [ZIM](https://openzim.org). They can be served using a ZIM server like Kiwix (the most common one), or [ZimReader](https://openzim.org/wiki/Zimreader), [GoZIM](https://github.com/akhenakh/gozim), & [others](https://openzim.org/wiki/Readers).

- [Kiwix.org ZIM archive list](https://wiki.kiwix.org/wiki/Content_in_all_languages)
- [Wikimedia.org ZIM archive list](https://dumps.wikimedia.org/other/kiwix/zim/wikipedia/)
- [List of ZIM BitTorrent links](https://gist.github.com/maxogden/70674db0b5b181b8eeb1d3f9b638ab2a)

ZIM archive dumps are usually published yearly, but the release schedule is not guaranteed. As of August 2019 the latest available dump containing all English articles is from October 2018:

**[`wikipedia_en_all_novid_2018-10.zim`](http://download.kiwix.org/zim/wikipedia_en_all_novid.zim)** (79GB, all English articles, no pictures/videos)

[`wikipedia_en_simple_all_novid_2019-05.zim`](https://dumps.wikimedia.org/other/kiwix/zim/wikipedia/wikipedia_en_simple_all_novid_2019-05.zim) (1.6GB, SimpleWiki English only, good for testing)

**Download your chosen Wikipedia ZIM archive** (e.g. `wikipedia_en_all_novid_2018-10.zim`)

```bash
mkdir -p /opt/wiki/data/dumps && cd /opt/wiki/data/dumps

# Download via BitTorrent:
transmission-cli --download-dir . 'magnet:?xt=urn:btih:O2F3E2JKCEEBCULFP2E2MRUGEVFEIHZW'

# Or download via HTTPS from one of the mirrors:
wget -c 'https://dumps.wikimedia.org/other/kiwix/zim/wikipedia/wikipedia_en_all_novid_2018-10.zim'
wget -c 'https://ftpmirror.your.org/pub/kiwix/zim/wikipedia/wikipedia_en_all_novid_2018-10.zim'
wget -c 'https://download.kiwix.org/zim/wikipedia/wikipedia_en_all_novid_2018-10.zim'

# Optionally after download, verify the length (fast) or MD5 checksum (slow):
stat --printf="%s" wikipedia_en_all_novid_2018-10.zim | grep 83853668638
md5sum wikipedia_en_all_novid_2018-10.zim | openssl dgst -md5 -binary | openssl enc -base64 | grep 01eMQki29P9vD5F2h6zWwQ
```

### XML Database Dump

- [WikiData.org Dump Types (JSON, RDF, XML)](https://www.wikidata.org/wiki/Wikidata:Database_download)
- [List of Dumps (XML dumps)](https://meta.wikimedia.org/wiki/Data_dump_torrents#English_Wikipedia)
- [List of Mirrors (XML dumps)](https://dumps.wikimedia.org/mirrors.html)

Database dumps are usually published monthly.  As of August 2019, the latest dump containing all English articles is from July 2019:

 **[`enwiki-20190720-pages-articles.xml.bz2`](https://meta.wikimedia.org/wiki/Data_dump_torrents#English_Wikipedia)** (15GB, all English articles, no pictures/videos)

[`simplewiki-20170820-pages-meta-current.xml.bz2`](http://itorrents.org/torrent/B23A2BDC351E58E041D79F335A3CF872DEBAE919.torrent) (180MB, SimpleWiki only, good for testing)

**Download your chosen Wikipedia XML dump** (e.g. `enwiki-20190720-pages-articles.xml.bz2`)

```bash
mkdir -p /opt/wiki/data/dumps && cd /opt/wiki/data/dumps

# Download via BitTorrent:
transmission-cli --download-dir . 'magnet:?xl=16321006399&dn=enwiki-20190720-pages-articles.xml.bz2'

# Download via HTTP:
# lol no. no one wants to serve you a 15GB file via HTTP
```

---

## Method #1: Run a caching proxy in front of Wikipedia.org

> <span style="color:#444">**Complexity:**</span> <span style="color:green">Low</span>  
> Minimal setup and operations requirements, no download of large dumps needed.  
> <span style="color:#444">**Disk space requirements:**</span> <span style="color:orange">On-Demand</span>  
> Disk is only used as pages are requested (can be 1gb up to 2TB+ depending on usage).  
> <span style="color:#444">**CPU requirements:**</span> <span style="color:green">Very Low</span>  
> Lowest out of the three options, can be run on a tiny VPS or home-server.  
> <span style="color:#444">**Content freshness:**</span> <span style="color:green">Very Fresh</span>  
> Configurable to cache content indefinitely or pull fresh data for every request.  

### a. Running with Nginx

Set the following options in your `/opt/wiki/.env` config file:
  `UPSTREAM_HOST=wikipedia.org`
  `UPSTREAM_WIKI=en.wikipedia.org`
  `UPSTREAM_MEDIA=upload.wikimedia.org`

Then run all the setup steps below under [Nginx Reverse Proxy](#) to set up Nginx.

Then restart nginx to apply your config with `systemctl restart nginx`.

Your mirror should now be running and proxying requests to Wikipedia.org!

Visit https://en.yourdomainhere.com to see it in action (e.g. https://en.wiki.example.com).

### b. Running with Caddy

Alternatively, check out a similar setup that uses Caddy instead of Nginx as the reverse proxy: https://github.com/CristianCantoro/wikiproxy

---

## Method #2: Serve the static HTML ZIM archive with Kiwix

> <span style="color:#444">**Complexity:**</span> <span style="color:orange">Moderate</span>  
> Static binary makes it easy to run, but it requires downloading a large dump file.  
> <span style="color:#444">**Disk space requirements:**</span> <span style="color:green">&gt;80GB</span>  
> The ZIM archive is a highly-compressed collection of static HTML articles only.  
> <span style="color:#444">**CPU requirements:**</span> <span style="color:green">Very Low</span>  
> Low, especially with a CDN in front (more than a proxy, but less than a full server).  
> <span style="color:#444">**Content freshness:**</span> <span style="color:red">Often Stale</span>  
> ZIM archives are published yearly (ish) by Wikipedia.org.  

First download a ZIM archive dump like `wikipedia_en_all_novid_2018-10.zim` into `/opt/wiki/data/dumps` as described above.


### a. Running with Docker

Run `kiwix-serve` with docker like so:

```bash
docker run \
    -v '/opt/wiki/data/dumps:/data' \
    -p 8888:80 \
    kiwix/kiwix-serve \
    'wikipedia_en_all_novid_2018-10.zim'
```

Or create `/opt/wiki/docker-compose.yml` and run `docker-compose up`:
```yml
version: '3'
services:
  kiwix:
    image: kiwix/kiwix-serve
    command: 'wikipedia_en_all_novid_2018-10.zim'
    ports:
      - '8888:80'
    volumes:
      - "./data/dumps:/data"
```

### b. Running with the static binary

1. **Download the latest `kiwix-serve` binary for your OS & CPU architecture**

    Find the latest release for your architecture here and copy its URL to download it below:
    https://download.kiwix.org/release/kiwix-tools/

    ```bash
    cd /opt/wiki
    wget 'https://download.kiwix.org/release/kiwix-tools/kiwix-tools_linux-x86_64-3.0.1.tar.gz'
    tar -xzf 'kiwix-tools_linux-x86_64-3.0.1.tar.gz'
    mv 'kiwix-tools_linux-x86_64-3.0.1' 'bin'
    ```

2. **Run `kiwix-serve`, passing it a port to listen on and your ZIM archive file**

    ```bash
    /opt/wiki/bin/kiwix-serve --port 8888 /opt/wiki/data/dumps/wikipedia_en_all_novid_2018-10.zim
    ```

    Your server should now be running!

    Visit http://en.yourdomainhere.com:8888 to see it in action!

### Optional Nginx Reverse Proxy

Set the following options in your `/opt/wiki/.env` config file:
```bash
UPSTREAM_HOST=localhost:8888
UPSTREAM_WIKI=localhost:8888
UPSTREAM_MEDIA=upload.wikimedia.org
```

Then run all the setup steps below under [Nginx Reverse Proxy](#) to set up Nginx. To run nginx inside docker-compose next to Kiwix, see the [Run Nginx via docker-compose](#) section below.

Your mirror should now be running and proxying requests to `kiwix-serve`!

Visit https://en.yourdomainhere.com to see it in action (e.g. https://en.wiki.example.com).


---

## Method #3: Run a full MediaWiki server

> <span style="color:#444">**Complexity:**</span> <span style="color:red">Very High</span>  
> Complex multi-component setup with an intricate setup process and high resource use.  
> <span style="color:#444">**Disk space requirements:**</span> <span style="color:red">&gt;550GB (>2TB needed for import phase)</span>  
>  The uncompressed database is very large (multiple TB with revision history and stubs).  
> <span style="color:#444">**CPU requirements:**</span> <span style="color:orange">Moderate (very high during import phase)</span>  
>  Depends on usage, but it's the most demanding out of the 3 options.  
> <span style="color:#444">**Content freshness:**</span> <span style="color:green">Very fresh</span>  
> Udpated database dumps are published monthly (ish) by Wikipedia.org.  

First download a database dump like [`enwiki-20190720-pages-articles.xml.bz2`](magnet:?xl=16321006399&dn=enwiki-20190720-pages-articles.xml.bz2&xt=urn:tree:tiger:zpqgda3rbnycgtcujwpqi72aiv7tyasw7rp7sdi&xt=urn:ed2k:3b291214eb785df5b21cdb62623dd319&xt=urn:aich:zuy4dfbo2ppdhsdtmlev72fggdnka6ch&xt=urn:btih:9f08161276bc95ec594ce89ed52fe18fc41168a3&xt=urn:sha1:54cbdd5e5d1ca22b7dbd16463f81fdbcd6207bab&xt=urn:md5:9be9c811e0cc5c8418c869bb33eb516c&tr=udp%3a%2f%2ftracker.openbittorrent.com%3a80&as=http%3a%2f%2fdumps.wikimedia.freemirror.org%2fenwiki%2f20190720%2fenwiki-20190720-pages-articles.xml.bz2&as=http%3a%2f%2fdumps.wikimedia.your.org%2fenwiki%2f20190720%2fenwiki-20190720-pages-articles.xml.bz2&as=http%3a%2f%2fftp.acc.umu.se%2fmirror%2fwikimedia.org%2fdumps%2fenwiki%2f20190720%2fenwiki-20190720-pages-articles.xml.bz2&as=https%3a%2f%2fdumps.wikimedia.freemirror.org%2fenwiki%2f20190720%2fenwiki-20190720-pages-articles.xml.bz2&as=https%3a%2f%2fdumps.wikimedia.your.org%2fenwiki%2f20190720%2fenwiki-20190720-pages-articles.xml.bz2&as=https%3a%2f%2fftp.acc.umu.se%2fmirror%2fwikimedia.org%2fdumps%2fenwiki%2f20190720%2fenwiki-20190720-pages-articles.xml.bz2&as=https%3a%2f%2fdumps.wikimedia.org%2fenwiki%2f20190720%2fenwiki-20190720-pages-articles.xml.bz2) into `/opt/wiki/data/dumps` as described above.

If you need to decompress it, `pbzip2` is much faster than `bzip2`:
```bash
pbzip2 -v -d -k -m10000 enwiki-20190720-pages-articles.xml.bz2
# -m10000 tells it to use 10GB of RAM, adjust accordingly
```

### a. Running with XOWA in Docker

https://github.com/QuantumObject/docker-xowa

```bash
docker run \
    -v /opt/wiki/data/xowa:/opt/xowa/ \
    -p 8888 \
    sblop/xowa_offline_wikipedia
```
```yaml
version: '3'
services:
  xowa:
    image: sblop/xowa_offline_wikipedia
    ports:
      - 8888:80
    volumes:
      - './data/xowa:/opt/xowa'
```

### b. Running with MediaWiki in Docker

- https://hub.docker.com/_/mediawiki
- https://github.com/wikimedia/mediawiki-docker
- https://github.com/AirHelp/mediawiki-docker
- https://en.wikipedia.org/wiki/MediaWiki
- https://www.mediawiki.org/wiki/MediaWiki
- https://www.mediawiki.org/wiki/Download
- https://www.wikidata.org/wiki/Wikidata:Database_download
- https://dumps.wikimedia.org/backup-index.html


**Configure your `docker-compose.yml` file**

Default MediaWiki config file: https://phabricator.wikimedia.org/source/mediawiki/browse/master/includes/DefaultSettings.php

Create the following `/opt/wiki/docker-compose.yml` file then run `docker-compose up`:
```yml
version: '3'
services:
  database:
    image: mariadb
    command: --max-allowed-packet=256M
    environment:
      MYSQL_DATABASE: wikipedia
      MYSQL_USER: wikipedia
      MYSQL_PASSWORD: wikipedia
      MYSQL_ROOT_PASSWORD: wikipedia
      
  mediawiki:
    image: mediawiki
    ports:
      - 8080:80
    depends_on:
      - database
    volumes:
      - './data/html:/var/www/html'
      # After initial setup, download LocalSettings.php into ./data/html
      # and uncomment the following line, then docker-compose restart
      # - ./LocalSettings.php:/var/www/html/LocalSettings.php
```


**Then import the XML dump into the MediaWiki database:**
- https://www.mediawiki.org/wiki/Manual:Importing_XML_dumps
- https://hub.docker.com/r/ueland/mwdumper/
- https://www.mail-archive.com/wikitech-l@lists.wikimedia.org/msg02108.html
    
**Do not attempt to import it directly with `importDump.php`, it will take months:**
```bash
php /var/www/html/maintenance/importDump.php enwiki-20170320-pages-articles-multistream.xml
```

**Instead, convert the XML dump into compressed chunks of SQL then import individually:**

*Warning: For large imports (e.g. English) this process can still take 5+ days depending on the system.*

```bash
apt install -y openjdk-8-jre zstd pbzip2

# Download patched mwdumper version and pre/post import SQL scripts
wget "https://github.com/pirate/wikipedia-mirror/raw/master/bin/mwdumper-1.26.jar"
wget "https://github.com/pirate/wikipedia-mirror/raw/master/preimport.sql"
wget "https://github.com/pirate/wikipedia-mirror/raw/master/postimport.sql"

DUMP_NAME="enwiki-20190720-pages-articles"

# Decompress the XML dump using all available cores and 10GB of memory
pbzip2 -v -d -k -m10000 "$DUMP.xml.bz2"

# Convert the XML file into a SQL file using mwdumper
java -server \
    -jar ./wikipedia-importing-tools/mwdumper-1.26.jar \
    --format=sql:1.5 \
    "$DUMP.xml" \
> wikipedia.sql

# Split the generated SQL file into compressed chunks
split --additional-suffix=".sql" --lines=1000 wikipedia.sql
for partial in $(ls *.sql); do
    zstd -z $partial
done

# Fix a schema issue that may otherwise cause import bugs
docker-compose exec database \
    mysql --user=wikipedia --password=wikipedia --database=wikipedia \
        "ALTER TABLE page ADD page_counter bigint unsigned NOT NULL default 0;"

# Import the compressed chunks into the database
for partial in $(ls *.sql.zst); do
    zstd -dc preimport.sql.zst $partial postimport.sql.zst \
    | docker-compose exec database \
        mysql --force --user=wikipedia --password=wikipedia --database=wikipedia
done
```

<sup>Credit for these steps goes to https://github.com/wayneworkman/wikipedia-importing-tools.</sup>


### Optional Nginx Reverse Proxy

Set the following options in your `/opt/wiki/.env` config file:
```bash
UPSTREAM_HOST=localhost:8888
UPSTREAM_WIKI=localhost:8888
UPSTREAM_MEDIA=upload.wikimedia.org
```

Then run all the setup steps below under [Nginx Reverse Proxy](#) to set up Nginx. To run nginx inside docker-compose next to MediaWiki, see the [Run Nginx via docker-compose](#) section below.

Your mirror should now be running and proxying requests to your wiki server!

Visit https://en.yourdomainhere.com to see it in action (e.g. https://en.wiki.example.com).

---

## Nginx Reverse Proxy

You can optionally set up an Nginx reverse proxy in front of `kiwix-serve`, `Wikipedia.org`, or a `MediaWiki` server to add caching and HTTPS support.

Make sure the options in `/opt/wiki/.env` are configured correctly for the type of setup you're trying to achieve.

- To run nginx in front of `kiwix-serve` on localhost, set:
  `UPSTREAM_HOST=localhost:8888`
  `UPSTREAM_WIKI=localhost:8888`
  `UPSTREAM_MEDIA=upload.wikimedia.org`
- To run nginx in front of Wikipedia.org, set:
  `UPSTREAM_HOST=wikipedia.org`
  `UPSTREAM_WIKI=en.wikipedia.org`
  `UPSTREAM_MEDIA=upload.wikimedia.org`
- To run nginx in front of a MediaWiki server on localhost, set:
  `UPSTREAM_HOST=localhost:8888`
  `UPSTREAM_WIKI=localhost:8888`
  `UPSTREAM_MEDIA=upload.wikimedia.org`
- To run nginx in front of a docker container via docker-compose:
  *See [Run Nginx via docker-compose](#) section below.*

### Install LetsEncrypt and Nginx

```bash
# Install the dependencies: nginx and certbot
add-apt-repository -y -n universe
add-apt-repository -y -n ppa:certbot/certbot
add-apt-repository -y -n ppa:nginx/stable
apt update -qq
apt install -y nginx-extras certbot python3-certbot-nginx
systemctl enable nginx
systemctl start nginx
```

### Obtain an SSL certificate via LetsEncrypt
```bash
# Load your config values from step 4 into the environment, and create dirs
source /opt/wiki/.env
mkdir -p "$CONFIG_DIR" "$CACHE_DIR" "$CERTS_DIR" "$LOGS_DIR" 

# Get an SSL certificate and generate the Diffie-Hellman parameters file
certbot certonly \
    --nginx \
    --agree-tos \
    --non-interactive \
    -m "ssl@$LISTEN_HOST" \
    --domain "$LISTEN_HOST,$LISTEN_WIKI,$LISTEN_MEDIA"
openssl dhparam -out "$PROJECT_DIR/data/certs/$DOMAIN.dh" 2048

# Link the certs into your project directory
ln -s /etc/letsencrypt/live/$DOMAIN/fullchain.pem $PROJECT_DIR/data/certs/$DOMAIN.crt
ln -s /etc/letsencrypt/live/$DOMAIN/privkey.pem $PROJECT_DIR/data/certs/$DOMAIN.key
```

LetsEncrypt certs must be renewed every 90 days or they'll expire and you'll get "Invalid Certificate" errors. To have certs automatically renewed periodically, add a systemd timer or cron job to run `certbot renew`. Here's an example tutorial on how to do that:
    https://gregchapple.com/2018/02/16/auto-renew-lets-encrypt-certs-with-systemd-timers/

### Populate the nginx.conf template with your config
<!-- {% raw %} -->
```bash
# Load your config options into the environment
source /opt/wiki/.env


# Download the nginx config template
curl --silent \
    "https://github.com/pirate/wikipedia-mirror/raw/master/etc/nginx/nginx.conf.template" \
    > "$CONFIG_DIR/nginx.conf.template"

# Fill your config options into nginx.conf.template to create nginx.conf
envsubst \
    "$(printf '${%s} ' $(bash -c "compgen -A variable"))"\
    < "$CONFIG_DIR/nginx.conf.template" \
    > "$CONFIG_DIR/nginx.conf"
```
<!-- {% endraw %} -->

### Run Nginx via systemd
```bash
# Link the your nginx.conf into the system's default nginx config location
ln -s -f "$CONFIG_DIR/nginx.conf" "/etc/nginx/nginx.conf"

# Restart nginx to load the new config
systemctl restart nginx
```

Now you can visit https://en.yourdomainhere.com to see it in action with HTTPS!

For troubleshooting, you can find the nginx logs here:
  `/opt/wiki/data/logs/nginx.err`
  `/opt/wiki/data/logs/nginx.out`

### Run Nginx via docker-compose

Set the config values in your `/opt/wiki/.env` file to correspond to the docker container's hostname that you want to proxy, and tweak the directory paths to be the paths inside the container. e.g. for `mediawiki`:
```bash
UPSTREAM_HOST=mediawiki:8888`
UPSTREAM_WIKI=mediawiki:8888`
UPSTREAM_MEDIA=upload.wikimedia.org

CERTS_DIR=/certs
CACHE_DIR=/cache
LOGS_DIR=/logs
```

Then regenerate your `nginx.conf` file with `envsubst` as described in [Nginx Reverse Proxy](#Nginx-Reverse-Proxy) below.

Then add the `nginx` service to your existing `/opt/wiki/docker-compose.yml` file:
```bash
version: '3'
services:
    
  ...

  nginx:
    image: nginx:latest
    volumes:
      - ./etc/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./data/certs:/certs
      - ./data/cache:/cache
      - ./data/logs:/logs
    ports:
      - 80:80
      - 443:443
```

---

# Further Reading

- https://github.com/openzim/mwoffliner (archiving only, no serving)
- https://www.yunqa.de/delphi/products/wikitaxi/index (Windows only)
- http://www.nongnu.org/wp-mirror/ (last updated in 2014, [Dockerfile](https://github.com/futpib/docker-wp-mirror/blob/master/Dockerfile))
- https://github.com/dustin/go-wikiparse
- http://www.learn4master.com/tools/python-and-java-libraries-to-parse-wikipedia-dump-dataset
- https://dkpro.github.io/dkpro-jwpl/
- https://towardsdatascience.com/wikipedia-data-science-working-with-the-worlds-largest-encyclopedia-c08efbac5f5c
- https://meta.wikimedia.org/wiki/Data_dumps/Import_examples#Import_into_an_empty_wiki_of_a_subset_of_en_wikipedia_on_Linux_with_MySQL
- https://github.com/shimondoodkin/wikipedia-dump-import-script/blob/master/example-result.sh
- https://github.com/wayneworkman/wikipedia-importing-tools
- https://github.com/chrisbo246/mediawiki-loader
- https://dzone.com/articles/how-clone-wikipedia-and-index
- https://www.xarg.org/2016/06/importing-entire-wikipedia-into-mysql/
- https://dengruo.com/blog/running-mediawiki-your-own-copy-restore-whole-mediwiki-backup
- https://brionv.com/log/2007/10/02/wiki-data-dumps/
- https://www.evanjones.ca/software/wikipedia2text.html
- https://lists.gt.net/wiki/wikitech/160482
- https://helpful.knobs-dials.com/index.php/Harvesting_wikipedia
- https://github.com/pirate/ArchiveBox/wiki/Web-Archiving-Community
