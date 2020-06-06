What is this?
=============

This document contains information on some inner workings of the Apple Photos program. All the information here was compiled by me looking at the library folder, examining the sqlite databases, monitoring changes in files, hexdumping, etc. while playing around with my personal photos collection.

Do what you want with this, but keep in mind it is unofficial and solely based on my personal observations, therefore it is incomplete and some parts could be plain wrong.

Apple may change how their software works at any time and without notice.

## Start here

First stop is to navigate to your photos library which is usually located under `Pictures` in your home folder

    $ cd ~/Pictures/Photos \Library.photoslibrary

Depending on how old your library is and weather you've migrated from iPhoto or started using Photos straight away, the contents of the library might differ slightly. Some important ones are:

  - `database` This is where Photos stores it's sqlite3 databases with the most important one being `Photos.sqlite`
  - `originals` Yep, that's all your original, unmodified photos in there
  - `resources` This directory seems to hold information on quite a few things:
    * files from your iCLoud _Shared Albums_, both yours and other people's
    * edited photos and editing information
    * resized copies of original photos
    * audit logs
    * project files
    * other iCloud related files
  - `private` Looks like these are config, state and cache files. This dir also contains files related to other services in the Apple Photos ecosystem, like the `photoanalysisd` (you know, the one that always eats a 100% of your CPU exactly when you are in the middle of doing some important CPU intensive work...)

## Database

The `Photos.sqlite` database contains lots of useful information, let's poke around. Browsing the `Z_PRIMARYKEY` table reveals what I would refer to as *Entities*. Each record has a name, an id (`Z_ENT`), `Z_SUPER` which maps to the entity's parent and some cached counters. I assume these are the various classes in Photos code.

It's important to note the `Z_ENT` for an entity, as this is used in other tables to identify the type of a record. For example, on my mac, an `Album`'s id is `26`.

Other tables of interest are:

  - `ZGENERICASSET` Records in this table represent your photos
  - `ZGENERICALBUM` Album and folder records are stored here
  - `Z_XXASSETS` It's a many-to-many relationship table that links photos to albums. The `XX` part is a number that corresponds to the `Album` entity from the `Z_PRIMARYKEY` table, so in my case the name of the table is `Z_26ASSETS`.
  - `ZADDITIONALATTRIBUTES` contains metadata about your images, like dimensions, original filename, the import session, location data, etc.
  - `ZEXTENDEDATTRIBUTES` has some more juicy metadata, some of which seems to be pulled off of the _EXIF_, like aperture, shutter speed, iso, focal length, sample rate for videos, camera and lens make and model, flash info, metering mode and more location data
  - `ZCOMPUTEDATTRIBUTES` This one seems to contain magic numbers which I assume are used internally by Photos to further classify your images, like how well you framed your shot, or if it has nice bokeh effect, etc. All some complicated AI shit.

These _ATTRIBUTES_ tables have foreign keys that link to the corresponding photo (or asset) from the `ZGENERICASSET` table.

I've only had a short peek at other tables, but once you get familiar with the ones mentioned above, it should be quite straightforward to navigate your way around the database if you want to explore further.

## File locations

Identifying an image is done by looking it up in the `ZGENERICASSET` table. For example, a photo with `ZUUID` of `D3FE5399-FA6A-47B0-976F-E7CC39EEF747` and a `ZDIRECTORY` equal to `D` (note that the directory name is the first character of the uuid) will be available under `D/D3FE5399-FA6A-47B0-976F-E7CC39EEF747.jpg` (also look at `ZFILENAME` for the exact filename including the extension).

Apple Photos does non-destructive edits to your images and original copies are kept in the `originals` folder.

Edited copies of photos are available in the `resources/renders` folder along with a `plist` file for each photo (sharing the same `ZUUID`) describing the edits. Edit information in these plists is base64 encoded, but decoding and hexdumping results in unreadable gibberish. I assume it's a proprietary encoding.

Resized photo versions are available in the `resources/derivatives` folder. These smaller versions include the latest edits applied to the photo.

  - a medium sized version is available directly under `resources/derivatives`
  - a tiny version (thumbnail) is available in `resources/derivatives/masters`

To sum up, assuming we made some edits to our photo, the following files will be available to us:

  1. Original unmodified image: `~/Pictures/Photos Library.photoslibrary/originals/D/D3FE5399-FA6A-47B0-976F-E7CC39EEF747.jpg`
  2. Image with latest edits applied: `~/Pictures/Photos Library.photoslibrary/resources/renders/D/D3FE5399-FA6A-47B0-976F-E7CC39EEF747_suffix.jpg`
  3. Smaller version with latest edits applied: `~/Pictures/Photos Library.photoslibrary/resources/derivatives/D/D3FE5399-FA6A-47B0-976F-E7CC39EEF747_suffix.jpg`
  4. Tiny version with latest edits applied: `~/Pictures/Photos Library.photoslibrary/resources/derivatives/masters/D/D3FE5399-FA6A-47B0-976F-E7CC39EEF747_suffix.jpg`

*NOTE 1*: `_suffix` may be different for you and I haven't yet figured out how it's generated.

*NOTE 2*: If there were no edits done to the original photo, then pt. 2 from the list above will not be available.

### Shared files

Copies of images you have shared vie Photos are made in `resources/cloudsharing`. This is also where the files that have been shared by someone else are stored.

The locations structure is similar to the one for your ordinary photos:

  - `data/YOURPERSONID` contains folders for each shared album and inside each folder are the images for that album along with a plist file that contains the album's name and some other metadata.
  - `resources/derivatives/masters` contain resized versions of the images grouped in folders

`YOURPERSONID` folder can be obtained by reading the `personID` file.

Another interesting observation I made is that not all photos you share get immediately copied into the `cloudsharing` directory. For example, I noticed that the copying happened on demand, e.g. when you go and actually browse the files in the _Shared Album_ via Photos app.

## Albums

Albums are recorded in `ZGENERICALBUM`. I've noticed that this table contains records of multiple `Entity` types. I tend to keep my photos quite organised and I've accumulated a bunch of folders with subfolders and then albums in them.

Browsing that table, it quickly becomes obvious what the fields are used for:

  - `Z_ENT` refers to the `Entity`, like a `Folder`, a `CloudSharedAlbum`, `Album`, etc.
  - `ZTRASHEDSTATE` nope, Photos doesn't immediately delete your stuff, just marks it as deleted
  - `ZKEYASSET`, `ZXX_ASSET` and `ZXX_CUSTOMKEYASSET` is a reference to the cover photo on your album. `XX` here again is a reference to an entity record from `Z_PRIMARYKEY`
  - `ZTITLE`, `ZUUID` shouldn't need any explanation


The table describing the relationship between albums and photos/videos is `Z_XXASSETS`, where `XX` is again the id of your `Album` entity. The `Z_XXALBUMS` in this table is the foreign key to the `ZGENERICALBUM` record and `Z_XXASSETS` to the `Z_GENERICASSET` (`XX` in this case is the id of the `GenericAsset` entity). What is `Z_FOK_XXASSETS`? I don't know.