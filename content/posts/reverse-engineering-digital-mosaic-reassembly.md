+++
# title = 'Reverse Engineering zoomable images: See the big picture'
title = 'Learn how to reverse engineer zoomable images and reassemble the pieces back together'
tags = ['ruby', 'algorithms', 'imagemagick', 'reverse engineering', 'puzzle', 'problem solving']
date = 2023-11-11T16:21:00+01:00
draft = false
slug = 'learn-reverse-engineer-zoomable-images-reassemble-pieces'
+++

![Cover image of the article featuring the mosaic with Lesya Ukrainka. Slava Ukraini!](/reverse-engineering-digital-mosaic-reassembly/slavaukraini-mosaic.png)

### Disclaimer
>This article is for educational purposes only and aims to provide insights into reverse engineering an algorithm. It is intended to help readers learn algorithmic concepts. Readers are advised to comply with applicable laws and regulations when engaging in reverse engineering, and the author does not endorse any unauthorized or unethical use of reverse engineering techniques.

## Introduction

Hello, my name is Aidan. I work as a **Software Engineer** @ [HackerOne](https://hackerone.com), where I create and maintain our platform for hackers and companies like [General Motors, Spotify, Shopify, and others](https://www.hackerone.com/customers) to collaborate on making world a vulnerability-free place. In my free time, however, I experience hyperfocus periods during which I use to conduct research, work on pet projects, and engage in various activities.
Sometimes, I satisfy my curiosity by reverse engineering things to understand how they work. This allows me to grasp ideas, gain perspective, and analyze potential solutions. If I'm fortunate enough, I may even find ways to improve them.

## The problem

If, like me, you grew up when the internet was becoming popular and we didn't have fancy JavaScript, you probably remember that pictures used to load as a single piece. However, due to slow connections, they would load in chunks from top to bottom. But as internet speeds improved, this became less noticeable. Nowadays, a 2-megabyte picture loads and displays almost instantly, making it a distant memory of the past.

![The Simpsons Comic Book Guy GIF](https://media.tenor.com/5lbGbnViz4gAAAAd/the-simpsons-comic-book-guy.gif)

While browsing a website, I encountered an image that I wanted to save but couldn't. This was frustrating. I realized that the image was utilizing a technique resembling Deep Zoom. I observed that my browser was downloading the image in fragments, much like how Google Maps loads tiles according to X, Y coordinates and scale. The tiles are sequentially displayed, with the next tile loading immediately after the previous one.

>The Deep Zoom file format is very similar to the Google Maps image format where images are broken into tiles and then displayed as required. The tiling typically follows a quadtree pattern of increasing resolution of image (in other words twice the zoom and twice the resolution).

![Downloaded tiles](/reverse-engineering-digital-mosaic-reassembly/tile-1.png)

During the analysis, I noticed that each piece fills its respective side in the matrix.

```
1_1, 2_1, 3_1
1_2, 2_2, 3_2
1_3, 2_3, 3_3
```

Great! This means I can reunite them.

## Construct the URL to fetch the tiles for X and Y

I have devised a plan:

* Construct the URL to fetch the tiles for X and Y
* Algorithm to download the tiles -- *which was the most fun problem* 🎢
* Glue them together and save

For simplicity, I decided to use the `OpenURI` module, which allows calling the [`#read`](https://ruby-doc.org/stdlib-2.6.3/libdoc/open-uri/rdoc/OpenURI.html) method directly on the URI.

>#read - reads a content referenced by self and returns the content as string. The string is extended with OpenURI::Meta. The argument options is same as #open.

```ruby
require 'open-uri'

RESOURCE = 'https://openseadragon.github.io/example-images/'.freeze

# ....

def build_url(x, y)
  uri = URI.join(RESOURCE, "#{@filename}/#{@filename}_files/", fetch_tile(x, y))
  puts uri
  uri
end

def fetch_tile(x, y)
  "#{@scale}/#{x}_#{y}.jpg"
end
```

Let's check if the code above works, and if we can actually build the working URL.

```bash
[2] pry(main)> @filename = "duomo"
=> "duomo" # the directory name where the file is stored.
[4] pry(main)> @scale = 13 # zoom factor
=> 13
# let's run the above-defined method to check
# if it provides us with the functional URL.
[6] pry(main)> build_url(2,6)
https://openseadragon.github.io/example-images/duomo/duomo_files/13/2_6.jpg
=> #<URI::HTTPS https://openseadragon.github.io/example-images/duomo/duomo_files/13/2_6.jpg>
```

Verifying the validity of the address and the existence of the image, and voila!

![Duomo tile - 2_6](/reverse-engineering-digital-mosaic-reassembly/2_6.jpg)

But wait, no volia yet. It's just the beginning.

## Algorithm to Download Tiles

Now that we have confirmed the functionality of the code above, we can incorporate it into our algorithm to download all the image tiles. This is an exciting part of the problem. While some people find joy in solving complex development tasks, participating in math olympics, or playing chess, reverse engineering can be equally addictive. Moreover, it can be profitable and help improve the world by preventing blackhat hackers from compromising your or your customers' platform.

In a nutshell, understanding how certain mechanisms work is always beneficial, as it enhances your expertise in your field! So with that said, let's write some code.

The coordinates `x` and `y` start at **0**.

```ruby
@horizontal = @vertical = 0
```

We define the array where we are going to store all our pieces that we have to glue back together.

```ruby
def download
  array = []

  loop do
    array[@vertical] = [] unless array[@vertical]
    array[@vertical] << build_url(@vertical, @horizontal).read

    # ...

    @horizontal += 1
  end

  array
end
```

We increment `@horizontal` by one for each downloaded horizontal piece. This causes it to scroll horizontally from `1_1` to `1_2`, `1_3`, and write it to the `array[@vertical]` row of the image.

```bash
[9] pry(main)> download
https://openseadragon.github.io/example-images/duomo/duomo_files/11/1_1.jpg
# ...
https://openseadragon.github.io/example-images/duomo/duomo_files/11/1_6.jpg
OpenURI::HTTPError: 404 Not Found
from ██████████████████████████████/open-uri.rb:387:in `open_http'
```

However, once we encounter a non-existent tile, it will trigger a 404 error. This indicates that we have finished processing the row, and we can now move on to the next line by incrementing `@vertical` by 1.
At the same time, we have to set `@horizontal` to the beginning, back to `0`
Let's add this piece of logic.

```ruby
def download
  array = []

  loop do
    begin
      array[@vertical] = [] unless array[@vertical]
      array[@vertical] << build_url(@vertical, @horizontal).read
    rescue OpenURI::HTTPError
      @vertical += 1
      @horizontal = 0

      retry
    end

    @horizontal += 1
  end

  array
end
```

But, the current issue is that it doesn't have a stopping mechanism. As a result, it continuously attempts to load the piece. Whenever it encounters an `HTTPError`, it increases the `@vertical` value, leading to an infinite loop as shown below.

```bash
[11] pry(main)> download
https://openseadragon.github.io/example-images/duomo/duomo_files/11/1_1.jpg
# ...
https://openseadragon.github.io/example-images/duomo/duomo_files/11/43_1.jpg
^CInterrupt:
```

To avoid this issue, we can check the array and break it if it's empty.

```ruby
def download
  # ...

  loop do
    begin
      # ...
    rescue OpenURI::HTTPError
      if array[@vertical].length.zero?
        array.delete_at(@vertical)
        break
      end
      # ...

      retry
    end

    # ...
  end

  array
end
```

## Glue them together and save

And now let's implement the logic to save the file. We will use the `Magick::ImageList` class from the `rmagick` library, which is exactly what we need in this case. We have an array of vertical stripes that we want to read using the `from_blob` method. We will push all of them to the `@image_list` and then use the `append` method to glue them together in the collection.

```bash
gem install rmagick
```

```ruby
def save
  @image_list = Magick::ImageList.new
  @tiles ||= download

  @tiles.map do |tiles|
    vertical_list = Magick::ImageList.new

    tiles.each do |tile|
      vertical_list.push(Magick::Image.from_blob(tile).first)
    end

    @image_list.push(vertical_list.append(true))
  end

  @image_list.append(false).write("#{@scale}_#{@filename}.jpg")
end
```

```bash
[2] pry(main)> save
https://openseadragon.github.io/example-images/duomo/duomo_files/10/0_0.jpg
# ...
https://openseadragon.github.io/example-images/duomo/duomo_files/12/14_0.jpg
=> JPEG:=>12_duomo.jpg JPEG 255x255=>3506x2570 255x255+0+0 DirectClass 8-bit 3054kb
```

The goal of this article is not to make you search the internet for images to combine. Instead, it aims to demonstrate that you can automate tasks by writing an algorithm and using appropriate tools. I utilized Ruby, the ImageMagick library, and the browser as my preferred tools. If you're new to programming or automation, it's exciting to discover the limitless possibilities of what you can achieve with a computer.

Developing websites is enjoyable, but it is not the only option available. As a problem solver, you can use programming languages to simplify your life and break free from monotonous routines. We have advanced from basic algorithms to AI that can assist you in correcting articles, improving your grammar, translating videos while preserving the speaker's voice, and many more, which makes it absolutely an incredible era. Thank you! Happy coding, everyone 🤖.
