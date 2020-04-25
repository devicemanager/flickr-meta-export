This is a fork of @nickivanov's flick-meta-export tool. Thanks for sharing @nickivanov!
I just wrote some more on this readme, no more. 

Also https://www.flickr.com/help/forum/en-us/72157711435626268/ has some hints on exiftool usage.
Specially liked:
Always regarded exiftool as an awesome tool, but I learned that it can reorganize images in directories constructed from EXIF tags, which makes it possible to structure local photo archives in directories as:

camera-model/year/month

e.g. "Canon EOS 7D\2009\10\4033316069_9206a84b49_o.jpg"

This is the command, if anyone is interested.

exiftool -r -d "%Y/%m" "-directory<NOMODEL/$FileModifyDate" "-directory<${model;}/$FileModifyDate" "-directory<NOMODEL/$DateTimeOriginal" "-directory<${model;}/$DateTimeOriginal" "C:\Pictures\Flickr\"

Multiple similar constructs take care of missing model tag or missing DateTimeOriginal tag. Those without a camera model or without EXIF will be in NOMODEL/file-modify-date directory. 

A day directory may be added with "%Y/%m/%d". 


Flickr allows you to download your photos and metadata. Unfortunatley, the metadata
is not stored along with the images as EXIF, XMP, IPTC or other standard format; instead
it will be downloaded as a series of JSON files, one per image.

These instructions work only for a POSIX compliant shell like on macOS, linux and perhaps cygwin.
Make sure to test it before doing a massive update on your images. 
If you are not familiar with command line tools, perhaps it's best to stay away from this. 
But since you found this on github, you might know what you are doing.

This tool allows you to convert all these JSON files into a single CSV file according to the supplied mapping. 
The CSV file can then be used by [ExifTool](https://www.sno.phy.queensu.ca/~phil/exiftool/) 
to update image EXIF/XMP/whatever metadata.

From the Flickr **Settings** page click **Request my Flickr data**. In a day or two you
will receive an email with two links, one for the metadata archive and another for
the image archive.

Extract the two archives, which will create two directories, one containing metadata
and another containing images.

The process might look like this:

0. Study the content of your Downloads

Before you begin you might want to study what has been downloaded from Flicr.
When I did this, I cleaned my Downloads folder and had only the downloads for flickr.
These instructions need to be adopted to your situation, hopefully it is understandable.
Unzipping all files in the download directory creates some structure, this is fine.

I started looking for all json files first:
find Downloads -type file -print| xargs -I{} -n 1 echo "{}" |grep json|less
Then looking for jpg 
find Downloads -type file -print| xargs -I{} -n 1 echo "{}" |grep json|less
Now there were maybe many files that are not jpg. This is how to find them
find Downloads -type file -print| xargs -I{} -n 1 echo "{}" |grep -v json|grep -v jpg|less
So I ended up having jpg, png mov and gif files. 
Adding a wc -l instead of less will show the amount of files.
The reason for having the xargs there will come next. 
Since the script and exiftool needs everything in the same directory, 
you could handle the update tag handling in in my case 4 steps each for a different filetype. 
find Downloads -type file -print|grep jpg| wc -l

I made a mistake and copied all json files to a photos directory, then cloned this repo,
and thought I could move the json files there. mv *.json flickr-meta-export/, 
but this resulted in -bash: /bin/mv: Argument list too long.
This did the trick: find photos -type file -print|grep json| xargs -I{} -n 1 rm "{}".
Replace rm which echo, like about, because it ruthless deletes all files.

Check the command
find Downloads -type file -print|grep photo_|grep json| xargs -I{} -n 1 echo "{}" photos/flickr-meta-export/|less
Check the count
find Downloads -type file -print|grep photo_|grep json| xargs -I{} -n 1 echo "{}" photos/flickr-meta-export/|wc -l
(If there are many photos_comment json files you get a difference, but this gives some confirmation on the volume)
Finally copy the json files to the destination
find Downloads -type file -print|grep photo_|grep json| xargs -I{} -n 1 cp "{}" photos/flickr-meta-export/

And do so for one of the media types (check the command like for json files and adjust)
find Downloads -type file -print|grep jpg| xargs -I{} -n 1 cp "{}" photos/flickr-meta-export/

For me this took quite some time so i went into the directory and used du -sch .. to see some progress.
So I further installed glances, using pip3 install glances. Then running glances showed resource usage.
Like io-rate on my disks: disk2 34.9M  35.6M, first read second write. 
I had 58GB data, and did not wait for it to finish  

Following the instructions below was a bit tricky since I used several different camera models and all store different tags.
So I first need to sort all images by camera type. 
To achieve this I needed to create a very large but simple text file for looking inside the jpg files. 
exiftool flickr-meta-export/ > exifdata.txt where flickr-meta-export was my directory contoining all jpg's.
Now I could look what data I had and see all the different tags for each camera model. There was no standard way of storing this meta-data.
less exifdata.txt
grep -e flickr-meta-export  -e Date exifdata.txt |less
grep -e flickr-meta-export -e Make -e Model exifdata.txt |less

Need to save this now and come back later. It's quite complicated because many images are missing different EXIF data, and writing this from all the json files could become a bigger mess.

1. Update tag mapping (optional).

You can update the mapping of Flickr metadata properties to EXIF/XMP/IPTC image 
tags if you want. A sample mapping file, `map.json` is provided. Its `input_tags`
properties contains a JSON array of strings that correspond to the Flickr metadata
properties, and the `output_tags` array contains the image tags to be set. Output 
tags can contain group name, e.g. `"EXIF:Make"`, following the ExifTool tag syntax.

Example input tag specifications and JSON they match:

- `"name"`
   Matches `{"name": "foobar"}`

- `"exif.Make"`
   Matches `{"exif": {"Make": "samsung"}}`

- `"albums[0].name"`
   Matches the `name` property of the first `albums` element, that is, will return
   `"album1"` given the following input: 
   `{"albums": [{"name": "album1"}, {"name": "album2"}]}

- `"tags[*].tag"`
   Matches the `tag` property of all `tags` element then joins them in a single
   string, separated by spaces. This will return `"foo bar"` given the following input: 
   `{"tags": [{"tag": "foo"}, {"tag": "bar"}]}

- `"exif.Make=foobar"`
   Supplies the default value for `"exif.Make"`, that is, will return `"foobar"`
   if there is no `Make` in `exif` or if there's no `exif`.


2. Extract Flickr metadata.

Flickr metadata does not follow any standard image metadata format, and I'm 
using Phil Harvey's [ExifTool](https://www.sno.phy.queensu.ca/~phil/exiftool/) to
update image files. As the first step I convert Flickr metadata into a format
ExifTool can consume -- a CSV file. This is where the Python program comes in.

    PYTHONIOENCODING=utf-8 \
    IMG_DIR=/path/to/image/files TAG_MAP=./map.json \
    ./meta2csv.py "/path/to/metadata/files/photo_*json" \
    >/path/to/image/files/meta.csv

Note that you need to put the metadata file pattern in quotes to avoid the shell 
globbing.

For example:

```
PYTHONIOENCODING=utf-8 \
IMG_DIR=/Volumes/media/downloads/data-download-1 TAG_MAP=./map.json \
./meta2csv.py "/Volumes/media/downloads/flickr-export/meta/photo_*json" \
>/Volumes/media/downloads/data-download-1/meta.csv
```

Two environment variables that control the program behaviour are:

- `IMG_DIR`: Indicates the directory path where the actual image files have been
  extracted from the Flickr export. If not specified the current directory is assumed.

- `TAG_MAP`: Points to the customized tag map file. If not specified, the default
  mapping hard-coded in the program is used.

Set `PYTHONIOENCODING` if any of your Flickr metadata -- image titles, descriptions,
tags etc. -- contain non-ASCII characters, otherwise Python stdout redirection 
will fail to print such characters.

3. Update image metadata with ExifTool.

Once the Flickr metadata is saved in a CSV file that can be understood by ExifTool,
you can update the image files. The command below will read the CSV file created
in the previous step, update each matching file's metadata with tags from that 
CSV file, and copy the file into a directory named after the Flickr album it belongs to.

It should run from the directory where image files have been extracted. 

    /path/to/exiftool -csv=<CSV file name> -e jpg -e png .

4. Separate images into albums.

Optionally you can now sort your images into directories based on the Flickr albums 
they were in. Note that if any of your images belongs to multiple albums on Flickr,
only the first album will be used to set the image metadata in step 2 above.

    /path/to/exiftool -'Directory<Album' -o %f.%e -e jpg -e png .

Note that this command will create a new subdirectory for each Flickr album; make sure 
that you run it from the directory where you want these album subdirectories to 
be created.
