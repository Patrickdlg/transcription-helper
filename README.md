# Transcription Helper

A simple program that lets you select a start + endpoint on a youtube video so you can isolate that clip for transcribing. This repo builds and deploys to github pages. You can [try it out here.](https://patrickdlg.github.io/transcription-helper/)

Features:
  - load youtube video
  - specify start and end time for loop
  - specify play speed
  - specify delay between loops

## Example

[Watch Example Video](mclean-example.mp4)

## Developing

```bash
npm run dev

# or start the server and open the app in a new browser tab
npm run dev -- --open
```

## Building

To create a production version of your app:

```bash
npm run build
```

You can preview the production build with `npm run preview`.