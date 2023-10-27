---
title: "Puzzle SVGs"
date: "2018-08-24"
categories: 
  - "miscellaneous"
  - "web-development"
tags: 
  - "design"
  - "puzzle"
  - "svg"
  - "web"
coverImage: "featured_image.png"
---

I keep meaning to improve my ability with Adobe Illustrator and i thought of a useful exercise that I could do to create something relatively useful. I want to create some puzzle pieces which can be used within Web Design to represents components of a whole. But have the puzzle pieces actually fit together, rather than all be the same.

So i hopped onto Youtube and saw a [great video](https://www.youtube.com/watch?v=fGixlPetcGI&index=2&list=LLE_IzGrXcKdm7XbmXRexaCg&t=0s) on how to create a puzzle board.
![](/images/4-Piece.png)

You'll notice that I deleted all junk sub-layers until i was just left with the four piece. I then saved the image as an SVG for web and played around changing the properties to test that i can change the colour of each piece.
![](/images/save-options.png)

I opened the `.svg` image within Notepad++ and changed the colour of each piece.

![](/images/colour_change.png)

Next, I tried to break each piece apart and use it within its own SVG image but it became so fiddely it wasnt really manageable. Each puzzle piece isnt just a quarter of the original puzzle piece. The little connectors squew the dimensions which makes the viewboxes difficult to calculate and maintain. Instead, i decided to take a simpler approach and have five total SVG images, one of the puzzle as a whole and one of each of the puzzle pieces. I simply took each piece within illustrator, copied it to a new .AI page and then _saved as_ SVG each piece.

Click on the sub layer puzzle piece, click the layer dropdown and select **Enter Isolation Mode**
![](/images/enter-isolation-mode.png)

Copy and paste the piece into a new illustrator page. Then go to Object->Artboards->Fit to Artwork Bounds
![](/images/fit_to_artwork_bounds.png)

This will nicely wrap the canvas around the piece.
![](/images/fitted_page.png)

Then save as an SVG and repeat for each puzzle piece. You can download a zip file containing all pieces and .ai files from here: [4 Piece Puzzle](/assets/misc/4-Piece-Puzzle.zip)

I also made a [3 Piece Puzzle](/assets/misc/3-Piece-Puzzle.zip)

![](/images/3-Piece.png)