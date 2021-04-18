---
layout: default
title: Aung San Su Kyi ANSCII image
date: '2021-04-18 21:34:13'
---

## Rust and me 
I have been learning about Rust during Thingyan Holidays to waste my time since the country is in dark times, we can't go out. I saw someone on reddit posting about anscii stich image and I found out no body have created ANSCII image for our leader [Aung San Su Kyi](https://en.wikipedia.org/wiki/Aung_San_Suu_Kyi). I found that there are already online tools to ANSCII image but most are not getting the result I wanted. So I decided to rewrite a simple ANSCII generator for ASSK image.

## How to 
Creating an ANSCII image is simple. 

1. Get an image 
2. Get gray scale of the image 
3. Convert pixels to ANSCII characters base on character mapping. 

### TLDR; 
Just read the codes. Codes are simple and anyone with little knowledge with Rust will understand.  [Code](https://github.com/ye-lin-aung/assk-anscii)

### Get an Image 

Getting image is easy. We can find any image online. I am using this image.
![Aung San Su Kyi](https://raw.githubusercontent.com/ye-lin-aung/assk-anscii/master/img/original.jpg)

Remember, we the ANSCII images are generated based on gray shades of the gray scaled image so its better to use images with black backgrounds or images with large amount of black colors are used. Too much bright colors such as bright yellow as the background on white foreground is not recommended.

### Reading Image 

I am using [image](https://crates.io/crates/image) crate for rust to read images. The crate have good documentation and well supported APIs. Since we only need reading the file into pixels, gray scaling image and writing images are needed, the image crate is enough for us. First we add image crate in Cargo.toml. 

```toml 
[dependencies]
image = "0.23.1"
```

And import the external crate at the top level of declaration file. such as `src/main.rs` 

```rust
extern crate image;
use image::GenericImageView;

//Rest goes down here
fn main() {} 
```

Now we can start reading the file. We  will open the file with image::open and exit the program if there is any problem with reading the file. 

```rust 
  let image = match image::open("assk.jpg"){

        Ok(image) => image,
        Err(error) => panic!("Problem reading image: {:?}", error)
    };
  println!("dimensions {:?}", image.dimensions());
```

### Gray Scalaing

We need to map each color pixel with a letter. Since images are with colors, we have RGBA profiles. Meaning you have (0,0,0,0) - (255,255,255,255) color variants. We don't want to map all these colors to letters, we don't have enough characters to represent all these different color pixels. So we need to gray scale the image. Gray scaling means turning each color pixel to a shade of gray. All the color variants will turned into different shades of gray. This will reduce from our 255^4  color variants to 255 shades of gray only. Its more easier to map them into characters now right?. We will now resize the image a bit so that image won't be so large viewing at terminal. Very large images will break the lines in terminal automatically and the result will be disoriented.

*Black and white won't work since changing the image to black and white will turn all bright pixels to white and dark pixels to black and areas of bright colors will just become a single white area which is not what we wanted* 

#### Gray scale vs Black and white 
![Gray Scale vs Black and White](/images/gs_vs_bl.jpg)

#### Getting grayscale of image and resizing by 5
```rust
    let gray_image = image.grayscale();

    let gray_image = gray_image.resize(gray_image.width() / 5 , gray_image.height() / 5, FilterType::Nearest);
```

### Mapping shades

```rust
    let character_set: [&str; 12] = [" ", "'", ",", ".", ":", ";", ".", "L", "O", "0", "#", "@"];
```
 
Now we only need to map the shades to characters. Mapping all 255 characters will give very detailed image. For us, we don't need a detail image. So we are going to use less character mapping. We will use 12 characters all 255 shades.  every 21 shades will be assigned with a letter. For example 0 - 21 is going to display as letter A, 21 - 42 as letter  B and goes on. We are going to use " " for lightest shade and "@" for darkest shade. Calculating the index of character for each shade is easy. We just need to divide the shade with 255 and multiple with the character map length - 1. So if we are given 255 (maximum shade), 255 / 255 * ( 12  - 1 )  = 11 meaning we are going to use the character from index 11 from character map. Since the number are between 0 - 255, it will only use range from 0 - 11 if we rounded the index so we don't need to worry about index out of bound issues. 


```rust 
    let mut art = String::new();
    let mut last_y = 0;
 
    for pixel in gray_image.pixels() {
       // Read the images and write new line if the last read y position is not same as new read y position 
        if last_y != pixel.1 {
            art.push_str("\n");
            last_y = pixel.1;
        }

        let pixel_data = pixel.2;

	     // We are going to convert RGB to single value shade.  
        let brightness:f64 = ((pixel_data[0] as u64 + pixel_data[1] as u64 + pixel_data[2] as u64) / 3) as f64;

        // Getting character position for shades, since we are using 12 characters, we will have a character for every 21 shades. 
        let character_position = ((brightness/255.0) * (character_set.len()  - 1) as f64 ).round() as usize;
         
         // Write the character to position
        art.push_str(character_set[character_position]);
    }
    fs::write("assk-black.txt", art.as_bytes()).unwrap();
``` 

Voila, this is how we get an ANSCII image generator written in Rust. [Code](https://github.com/ye-lin-aung/assk-anscii)

*You can play around the character map to get better results. Example some characters are more visible in dark backgrounds than some are in white. Using white spaces for darkest shades are good for dark backgrounds but not god for light backgrounds. The character map can be added new characters to get more detailed images and swap characters to get better image clarity based on your terminal background. You can see I use different character sets for different background images.* 

### Original 
![Original](https://raw.githubusercontent.com/ye-lin-aung/assk-anscii/master/img/original.jpg "Original")

### White Background
![White](https://raw.githubusercontent.com/ye-lin-aung/assk-anscii/master/img/white.jpg "White")

### Black Background
![Black](https://raw.githubusercontent.com/ye-lin-aung/assk-anscii/master/img/black.png "Black")

You will see the current generated image are a bit slender. This is due to font spacing. You can swap better characters to represent better image.
