---
title: "Interactive Paint Wall Exhibit"
date: 2023-07-20T20:57:05-04:00
cover:
  image: /projects/paint_wall.jpg
  hidden: true
featured: true
---

![A whole wall of paintings](/projects/paint_wall.jpg)

## [Launch interactive paint wall exhibit](https://varunm22.github.io/paint-wall)

Starting in January 2022, I made a paint wall through a bunch of paint nights with friends, and you can read more about that [here]({{< ref "/hobbies/painting/post-3-paint-wall" >}}). Whenever I've shown people the picture of the whole wall, they've always immediately zoomed in, trying to look closer at individual paintings, but the resolution of each painting is never great. 

Given that I have nice pictures of each individual painting (taken before putting on the wall), I've always wanted to build out an interactive version of the paint wall where you can click on paintings in the overall picture and see a larger version of the painting along with info like who painted it and any notes they may have had.

Overall, the implementation was pretty straightforward using HTML Image Maps. However, I didn't really want to manually define all of the corners of the individual paintings. Since the paintings are arranged in a fairly regular grid, I was able to write a python script to autogenerate the image map. This uses various defined constaints such as the pixel offsets of the painting grid corners from the corners of the image itself or even which column indices are horizontal or vertical. Check out the source code [here](https://github.com/varunm22/paint-wall).

I am still planning on adding features to this project, potentially including some way to see all paintings for a given artist. Any new implemented features will get added to this post for future reference!
