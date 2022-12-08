---
author: qlyoung
layout: post
title:  "music management"
---

1. Tx from source to local staging directory
2. `beet import` from staging directory into mounted remote share[0]


```
                 { source }
                     |
                     .
                    ---   (net) <1>
                     .
                     |
                     v
              [ local:music ]
                     |
                     .
            beet    ---   (net) <2>
           import    .
                     |
                     v
      [ remote:music => local:remote/music]
```

3. cron job on remote periodically copies new files into a sync directory,
   transcoding any lossless files to `-q6` ogg vorbis to reduce size[1]

```
             [ remote:music ]
                     |
                     |
    music-sync.sh    |    @ 2hr <3>
                     |
                     v
           [ remote:music-sync ]
```

4. sync directory shared to all devices via syncthing[2]

```
           [ remote:music-sync ]
                     |
                     |
                     .
        syncthing   ---   (net) <4>
                     .
                    ...
                   . . .
                  .  .  .
                 .   .   .
                /    |    \
               v     v     v
            phone  laptop  idk
```

[0] <https://beets.io/>

[2] <https://syncthing.net/>

[1]

```sh
#!/usr/bin/fish
set MUSICDIR "./music/"
set SYNCDIR  "./music-sync"

for dir in (find "$MUSICDIR" -type d | cut -d'/' -f3-)
    mkdir -p "$SYNCDIR/$dir"
end

for file in (find "$MUSICDIR" -type f -name '*.flac' -o -name '*.mp3' -o -name '*.ogg' | cut -d'/' -f3-)
    set ifile (echo "$MUSICDIR/$file")
    switch $file
    case "*.flac"
        set ofile (echo "$SYNCDIR/"(echo "$file" | sed "s/flac/ogg/"))
        if test -e "$ofile"
            echo "$ofile exists; skipping"
            continue
        end
        echo ">> Transcoding '$ifile' to '$ofile'"
        oggenc -q6 -o "$ofile" "$ifile"
    case "*"
        set ofile (echo "$SYNCDIR/$file")
        if test -e "$ofile"
            echo "$ofile exists; skipping"
            continue
        end
        echo ">> Copying '$ifile' to $ofile"
        cp "$ifile" "$ofile"
    end
end
```
