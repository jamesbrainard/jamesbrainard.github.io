---
layout: project
title:  "Web-Based Mini-Projects"
date:   2024-02-19 00:00:00 +0300
categories: jekyll update
image: "/assets/img/post_images/qr_code_generator.png"
---
They say that the best way to figure out what you like is to figure out what you don't like. - This is how I learned that I don't like web development!

*How do I make a button? Why is this container in the wrong place?*

Every facet of programming is tough to learn and issues are inevitable, but problem-solving in web development might be the most frustrating of all. It feels awfully good when things work, though.

# QR Code Generator
The first of these two mini-projects is a QR code generator. In retrospect, this project wasn't named perfectly - QR (Quick Response) codes are not necessarily "generated" as much as they are "translated." A QR code is the exact same no matter who or what translates a URL into the black-and-white pattern of squares.

My major criticism of current QR code generators is that they actually *do* generate QR codes. Rather than just giving people a QR code that goes straight to the link they request, many QR code generators reroute links through their own websites, allowing them to put time limits on users' QR codes and try to make them pay.

It's predatory marketing, and it's completely unnecessary.

In the spring of 2024, some friends needed to generate dozens of QR codes for Creighton's fall/spring break Service & Justice Trips, and they fell into this trap. Websites would lock a certain number of QR codes behind a paywall, force scanners through advertiser links, put time limits on codes, or some combination of the three.

The Javascript code for this project is actually pretty simple - so simple that I can paste all of it here:

{% highlight javascript %}
function generateQRCode() {
    const urlInput = document.getElementById("urlInput");
    const inputValue = urlInput.value;

    if (inputValue.trim() !== "") {
        const qrcodeContainer = document.getElementById("qrcode");
        qrcodeContainer.innerHTML = "";

        try {
            const qr = new QRious({
                value: inputValue,
                size: 200,
            });

            const qrCodeImg = document.createElement("img");

            qrCodeImg.src = qr.toDataURL();

            qrcodeContainer.appendChild(qrCodeImg);
        } catch (error) {
            console.error("Error generating QR code:", error);
            alert("Error generating QR code. Please check the console for details.");
        }
    } else {
        alert("Please enter a valid URL");
    }
}

document.getElementById("urlInput").addEventListener("keydown", function (event) {
    if (event.key === "Enter") {
        generateQRCode();
    }
});
{% endhighlight %}

This code makes use of QRious, a Javascript-based library literally made for generating QR codes. It was perfect for my needs. The web page definitely isn't the prettiest thing, but I find that it works, and that's enough! My friends have been using it ever since I put it on GitHub Pages.

# Morse Code Translator

This felt more like a "because I can" project, but it's still worth mentioning. This website takes as its input a series of characters in one box and returns the translation of those characters into morse code in the other box. Nothing crazy! I wanted to use a map and create type boxes somehow, and this was a good excuse. Overall, I found that it was a fun little challenge that I haven't truthfully found a whole lot of use for.

The code is even simpler than the QR code generator:

{% highlight javascript %}
function translateToMorse() {
    const text = document.getElementById('inputText').value.toUpperCase();
    const morseCode = text.split('').map(charToMorse).join(' ');
    document.getElementById('outputMorse').value = morseCode;
}

function charToMorse(char) {
    switch (char) {
        case 'A': return '.-';
        case 'B': return '-...';
        case 'C': return '-.-.';
        case 'D': return '-..';
        case 'E': return '.';
        case 'F': return '..-.';
        case 'G': return '--.';
        case 'H': return '....';
        case 'I': return '..';
        case 'J': return '.---';
        case 'K': return '-.-';
        case 'L': return '.-..';
        case 'M': return '--';
        case 'N': return '-.';
        case 'O': return '---';
        case 'P': return '.--.';
        case 'Q': return '--.-';
        case 'R': return '.-.';
        case 'S': return '...';
        case 'T': return '-';
        case 'U': return '..-';
        case 'V': return '...-';
        case 'W': return '.--';
        case 'X': return '-..-';
        case 'Y': return '-.--';
        case 'Z': return '--..';
        case '0': return '-----';
        case '1': return '.----';
        case '2': return '..---';
        case '3': return '...--';
        case '4': return '....-';
        case '5': return '.....';
        case '6': return '-....';
        case '7': return '--...';
        case '8': return '---..';
        case '9': return '----.';
        case ' ': return ' '; // space
        default: return ''; // ignore unknown characters
    }
}
{% endhighlight %}