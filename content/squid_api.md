+++
title = "On basic \"reverse-engineering\" of simple web API's"
date = 2025-04-19
[taxonomies]
tags = ["web"]
+++

So, you have a not actively hostile* website you want to automate some actions with, like a nice music downloader `squid.wtf`.
Let's write a simple script to use it from the terminal.

\* i.e. without captchas on every corner and stuff

## Recording the network traffic


{{ video(name="squid/initial_search_log.mp4") }}

To begin with, we need to record what the web page sends to the server and what responses it expects.
To do so, open [the webpage]( https://eu.qobuz.squid.wtf/ ) in your browser of choice. I'll use zen, but the same is applicable to pretty much any of them.
Then open the `devtools` on the `network` tab (`Ctrl+Shift+E` in my case, but you can also `Ctrl+Shift+I` and select the `network` tab, for example).
Finally, type some album name, for example, `Haken - Vector` into the search bar, select some version of it to download, and see the log being populated with requests the page makes.

## Making sense of the network log

{{ video(name="squid/unclip_devtools_find_request.mp4") }}

To make more space for the `devtools` window, you can unclip it from the browser by pressing `...` in its top right corner and choosing `Separate window` (or whatever you like). Then its a matter of selecting the queries that look promising. In my case, it's the second successful request (since the 1st one contained an extra dash and returned no albums).

To find the request I need, I look 
* at the `file` field in the response log -- `api/get-music?q=haken - vector&offset=0` seems important enough, while a randomly-looking .jpg is clearly some kind of an asset (a cover art in this case);
* at the `domain` field -- `eu.qobus.squid.wtf` is the site we're using\*\*, while `static.qobiz.com` also obviously stores assets;
* at the `response` and `request` (for POST requests only) tabs when the request is selected.

\*\* Technically, the webpage may use some other site's API, although it happens not too often.

Since this is a rather simple case, determining what's what is straightforward. The webpage used the following endpoints:
* `api/get-music?q=$query&offset=$offset`, where q is the query (`haken - vector`), and offset is pagination in the form of (`10 * N`), i.e. it gets you the next 10 albums in case you haven't found the one you were looking for on the `N - 1`th page (`limit` field for albums says 10);
* `api/get-album?album_id=$album_id` gets you info about the selected album and expects its id as input. The id is obtained from the results of the `get-music` call.
* `api/download-music?track_id=$track_id&quality=$quality` returns you a download link for the particular track_id (obtained from the `get-album` call). Right off the bat, I haven't noticed that exact value in the album info, so it can probably be left as 27 and delt with later in case it's not simply an internal way to refer to the best quality the track has (as opposed to something specific, like a 24-bit flac). 

## Scripting

When the process is more or less clear, those same requests can eqsily be performed from a script or a program (like a telegram bot or something).
I usually do this by opening 2 windows side-by-side, one with python REPL, another one with whatever I'm writing (in this example, a python script). 
REPL is used to debug logic, which is then rewritten and sewn together in the actual program.

The resulting PoC terminal wrapper could look something like this.

```python
import requests
import json
from urllib.parse import quote, urlencode

base_url = "https://eu.qobuz.squid.wtf/api/"

def search(terms: str, offset: int = 0):
    #https://get-music?q=haken%20-%20vector&offset=0
    query = base_url + "get-music?" + urlencode({
        "q": terms,
        "offset": 0
    },
    quote_via = quote)
    response = requests.get(query)
    if response.ok:
        data = json.loads(response.content)
        for item in data["data"]["albums"]["items"]:
            print(
                item["artist"]["name"],
                item["title"],
                (item["version"] or ""),
                item["id"]
            )

def getDownloadUrl(trackId: int, quality = 27):
    #https://download-music?track_id=52848233&quality=27
    query = base_url + "download-music?" + urlencode({
        "track_id": trackId,
        "quality": quality
    },
    quote_via = quote)
    response = requests.get(query)
    if response.ok:
        data = json.loads(response.content)
        url = data["data"]["url"]
        return url

def downloadAlbum(id: str):
    #https://get-album?album_id=rygp0c3z97ufb
    query = base_url + "get-album?" + urlencode({
        "album_id": id
    },
    quote_via = quote)
    response = requests.get(query)
    if response.ok:
        data = json.loads(response.content)
        trackList = data["data"]["tracks"]["items"]
        fields_to_select = ["track_number", "title", "version", "id"]
        track_info = [{field: track[field] for field in fields_to_select} for track in trackList]
        for track in track_info:
            response = requests.get(getDownloadUrl(track["id"]))
            if response.ok:
                filename = f"{track["track_number"]}. {track["title"]}{' (' + track["version"] + ')' if track["version"] else ""}.flac"
                with open(filename, "wb") as trackFile:
                    trackFile.write(response.content)
                print("Saved:", filename)


if __name__ == "__main__":
    print("Search: ", end = '')
    search(input())
    print("Enter the id (last value) of the album to download: ", end = '')
    downloadAlbum(input())
```

To make it more usable, there needs to be some proper error handling with retries, possibly making use of pagination, and tagging music (since the resulting files lack those), but it goes out of the scope of the write-up. Also, it's not what I needed it for :)
