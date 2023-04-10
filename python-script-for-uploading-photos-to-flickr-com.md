---
description: >-
  This Python script efficiently uploads JPG and PNG images from a specified
  directory and its subdirectories to Flickr.com. It utilizes an authorization
  token for secure access, displays a progress bar
---

# Python Script for Uploading Photos to Flickr.com

Install the required Python packages:

```sh
pip3 install flickrapi python-magic Pillow tqdm
```

Obtain your Flickr API key and secret by creating a new app on Flickr's website:

[https://www.flickr.com/services/apps/create/apply/](https://www.flickr.com/services/apps/create/apply/)

Create the Python script `flickr.py`:

```python
#!/usr/bin/env python3

import hashlib
import os
import sys
from datetime import datetime
from pathlib import Path

import magic
from PIL import Image, ExifTags
from flickrapi import FlickrAPI
from tqdm import tqdm

API_KEY = "63e341055a7526bea56054506d5b2969"
API_SECRET = "ecdd4a09f1fba2ec"


def authenticate(api_key: str, api_secret: str) -> FlickrAPI:
    home_directory = Path.home()
    oauth_token_path = home_directory / "oauth-tokens.sqlite"

    flickr = FlickrAPI(
        api_key, api_secret, format="etree", store_token=True, cache=oauth_token_path
    )

    if not flickr.token_valid(perms="write"):
        flickr.get_request_token(oauth_callback="oob")
        authorize_url = flickr.auth_url(perms="write")
        print("Go to the following URL to authorize the app:")
        print(authorize_url)

        verifier = input("Enter the verifier code: ")
        flickr.get_access_token(verifier)

    return flickr


def is_image(file_path: Path) -> bool:
    mime = magic.Magic(mime=True)
    file_mime = mime.from_file(str(file_path))
    return file_mime.startswith("image/") and file_path.suffix.lower() in (
        ".jpg",
        ".jpeg",
        ".png",
    )


def md5_hash(file_path: Path) -> str:
    hash_md5 = hashlib.md5()
    with file_path.open("rb") as f:
        for chunk in iter(lambda: f.read(4096), b""):
            hash_md5.update(chunk)
    return hash_md5.hexdigest()


def get_image_datetime(file_path: Path) -> datetime:
    try:
        with Image.open(file_path) as img:
            exif = {
                ExifTags.TAGS[k]: v
                for k, v in img._getexif().items()
                if k in ExifTags.TAGS
            }
            date_str = exif.get("DateTimeOriginal") or exif.get("DateTime")
            if date_str:
                return datetime.strptime(date_str, "%Y:%m:%d %H:%M:%S")
    except (AttributeError, KeyError, ValueError):
        pass
    return None


def get_set_id(flickr: FlickrAPI, set_name: str) -> str:
    sets = flickr.photosets.getList()["photosets"]["photoset"]
    for s in sets:
        if s["title"]["_content"] == set_name:
            return s["id"]
    new_set = flickr.photosets.create(title=set_name, primary_photo_id=photo_id)
    return new_set["id"]


def is_duplicate_image(flickr: FlickrAPI, file_path: Path) -> bool:
    file_md5 = md5_hash(file_path)
    response = flickr.photos.search(
        user_id="me", tags=file_md5, extras="original_format"
    )

    if response.attrib.get("stat") == "ok":
        photos = response.find(".//photos")
        if photos is not None and int(photos.attrib["total"]) > 0:
            return True
    return False


def get_relative_directory(input_path: Path, file_path: Path) -> str:
    relative_path = file_path.relative_to(input_path)
    if relative_path.parent != Path("."):
        return str(relative_path.parent)
    return ""


def main(flickr: FlickrAPI) -> None:
    while True:
        input_path = input(
            "Enter the directory path or file path of the images to upload (or type 'x', 'q', or 'exit' to quit): "
        )
        if input_path.lower() in ("exit", "x", "q"):
            break

        input_path = Path(input_path)

        if not input_path.exists():
            print("The specified path does not exist. Please try again.")
            continue

        if input_path.is_file():
            files = [input_path]
        else:
            files = [file for file in input_path.rglob("*.*") if is_image(file)]

        duplicate_files = []

        for idx, file_path in enumerate(files, start=1):
            file_name = file_path.name
            if is_duplicate_image(flickr, file_path):
                print(f"Skipping duplicate image: {file_name}")
                duplicate_files.append(file_name)
                continue

            image_datetime = get_image_datetime(file_path)
            tags = "uploaded_by_script"
            if image_datetime:
                tags += f" year{image_datetime.year} month{image_datetime.month}"

            relative_directory = get_relative_directory(input_path, file_path)
            if relative_directory:
                tags += f" relative_directory:{relative_directory}"

            try:
                response = flickr.upload(
                    filename=str(file_path), is_public=0, tags=tags
                )
                if response.attrib.get("stat") != "ok":
                    error_message = response.find(".//err").attrib
                    print(f"Failed to upload {file_name}: {error_message}")
                    continue
            except Exception as e:
                print(f"Error while uploading {file_name}: {e}")
                continue

            print(f"Uploaded image {idx}/{len(files)}: {file_name}")

        print(f"Finished uploading images from '{input_path}'")

        if duplicate_files:
            with open("duplicates.txt", "w") as f:
                f.write("\n".join(duplicate_files))
            print(f"Duplicate file names have been saved to 'duplicates.txt'.")
    while True:
        input_path = input(
            "Enter the directory path or file path of the images to upload (or type 'x', 'q', or 'exit' to quit): "
        )
        if input_path.lower() in ("exit", "x", "q"):
            break

        input_path = Path(input_path)

        if not input_path.exists():
            print("The specified path does not exist. Please try again.")
            continue

        if input_path.is_file():
            files = [input_path]
        else:
            files = [file for file in input_path.glob("*.*") if is_image(file)]

        duplicate_files = []

        for idx, file_path in enumerate(files, start=1):
            file_name = file_path.name
            if is_duplicate_image(flickr, file_path):
                print(f"Skipping duplicate image: {file_name}")
                duplicate_files.append(file_name)
                continue

            image_datetime = get_image_datetime(file_path)
            tags = "uploaded_by_script"
            if image_datetime:
                tags += f" year{image_datetime.year} month{image_datetime.month}"

            try:
                response = flickr.upload(
                    filename=str(file_path), is_public=0, tags=tags
                )
                if response.attrib.get("stat") != "ok":
                    error_message = response.find(".//err").attrib
                    print(f"Failed to upload {file_name}: {error_message}")
                    continue
            except Exception as e:
                print(f"Error while uploading {file_name}: {e}")
                continue

            print(f"Uploaded image {idx}/{len(files)}: {file_name}")

        print(f"Finished uploading images from '{input_path}'")

        if duplicate_files:
            with open("duplicates.txt", "w") as f:
                f.write("\n".join(duplicate_files))
            print(f"Duplicate file names have been saved to 'duplicates.txt'.")


if __name__ == "__main__":
    flickr = authenticate(API_KEY, API_SECRET)
    main(flickr)
```

Save the contents to a file named flickr.py.

Make the flickr.py script executable:

```sh
chmod +x flickr.py
```

To make the script globally executable, create a symlink to it in /usr/local/bin:

```sh
sudo ln -s /path/to/flickr.py /usr/local/bin/flickr
```

Now you can run the script from anywhere using the command flickr:

```sh
flickr
```
