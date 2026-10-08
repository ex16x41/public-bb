# Public Cloud Storage Discovery — Testing Notes

During testing of an authorized target application, browser traffic showed static assets loading from a public cloud-storage bucket.

## What I Tested

I took the observed asset path and manually tested higher-level directory paths.

Example pattern:

```text
/storage-bucket/path/to/asset.ext
↓
/storage-bucket/
```

## What Happened

The bucket root returned a public object listing.

This exposed:

- asset filenames
- project-related directories
- media files
- configuration-related objects
- several files with names suggesting financial/reporting data

A small number of those report files were anonymously readable and contained internal-looking operational and billing metadata.

## Finding Pattern

```text
Application loads public cloud asset
        ↓
Identify bucket name
        ↓
Test bucket root
        ↓
Public object listing exposed
        ↓
Review filenames only
        ↓
Sensitive-looking objects discovered
        ↓
Minimal anonymous-read verification
```

## Key Lesson

A public asset does not necessarily mean the **entire bucket** was intended to be public.

When an application loads assets from cloud storage, check whether:

```text
✓ bucket listing is enabled
✓ unrelated objects are exposed
✓ internal-looking reports are publicly readable
```

