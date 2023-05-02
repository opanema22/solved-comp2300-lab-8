Download Link: https://assignmentchef.com/product/solved-comp2300-lab-8
<br>
Before you attend this week’s lab, make sure:

<ol>

 <li>you can load &amp; store data to/from memory using <code>ldr</code> &amp; <code>str</code> with different <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/lectures/week-5/#offset-load-and-store-with-write-back">addressing modes</a></li>

 <li>you’ve completed the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/">Week 6/7 blinky lab</a></li>

</ol>

In this week’s lab you will:

<ol>

 <li>use a simple data structure for storing dot-dash “morse code” codepoints</li>

 <li>store multiple morse codepoints in an array-like structure in memory</li>

 <li>write a program to read an <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/ASCII">ASCII</a> string and “output” it as morse code blinks</li>

</ol>

<h2 id="introduction">Introduction</h2>

<a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Morse_code">Morse code</a> is a simple communication protocol which uses “dots” and “dashes” to represent the letters of the alphabet.

The dots and dashes can be represented in different ways—as dots or lines on a page, as short or long beeps coming out of a speaker, or <a class="acton-tabs-link-processed" href="https://hackaday.com/2018/04/13/another-reason-to-learn-morse-code-kidnapping/">hidden in a song on the radio to reach kidnap victims</a>, or as short or long “blinks” of the LED light on your discoboard. In this lab content the morse code will be represented visually using a sequence of <code>.</code> (dot) and <code>_</code> (dash) characters, but by the end of the lab you’ll be sending morse code signals by blinking the red LED on your discoboard in short (dot) and long (dash) bursts. Here’s the full morse alphabet (courtesy of <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/Morse_code">Wikipedia</a>)

<img decoding="async" alt="Morse code alphabet" data-src="https://cs.anu.edu.au/courses/comp2300/assets/labs/lab-8/morse-code.svg" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://cs.anu.edu.au/courses/comp2300/assets/labs/lab-8/morse-code.svg" alt="Morse code alphabet">

 </noscript>

<p class="talk-box">Discuss with your neighbour—have you ever seen (or even used!) morse code before? Where/when?

<h2 id="exercise-1-a-led-utility-library">Exercise 1: a LED utility library</h2>

Remember the setup code you wrote in the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/">blinky lab</a>? Now that you’ve got some more skills with <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/05-functions/">functions</a> under your belt, you can probably imagine how you might package up a lot of that <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/#load-twiddle-store">load-twiddle-store</a> stuff into functions to make the code a bit more readable (this was actually proposed as an extension exercise at the end of the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/05-functions/#led-library-extension">last lab</a>).

In fact, Ben Swift (another lecturer at ANU) already wrote this library for you (because he’s great &#x1f60a;). You can find it in the <code>lib/led.S</code> file after you fork &amp; clone the <a class="acton-tabs-link-processed" href="https://gitlab.cecs.anu.edu.au/comp2300/2021/comp2300-2021-lab-8">lab 8 template</a> from GitLab.

Have a read through the code in <code>led.S</code>—you should now be at the stage where you can look at assembly code like this and at least get a <em>general</em> sense of what it does and how it works. Here are a couple of things to pay particular attention to as you look over it.

<ul>

 <li>The code uses <code>push</code> (to store the value in a register onto the stack, and <em>decrement</em> the stack pointer <code>sp</code>) and <code>pop</code> (to load the top value on the stack into a register and <em>increment</em> the stack pointer <code>sp</code>). You can do this in other ways (e.g. <code>stmdb pc!, {lr}</code>) but <code>push</code> and <code>pop</code> are convenient when you want to want to use <code>sp</code> to keep track of the stack. You can see the <code>push</code>/<code>pop</code> instructions in <em>Section A7.7</em> of your your <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/v_media/manuals/ARMv7-M-architecture-reference-manual.pdf">ARMv7 reference manual</a></li>

 <li>Some (but not all) of the functions take arguments (described in the comments), so before you call these functions make sure you’ve got the right values in these registers to pass arguments to the functions.</li>

 <li>The <code>.global delay, red_led_init, red_led_on, red_led_off</code> line is necessary because you’re putting the code above into a separate file to the one where the rest of your program will be (<code>src/main.S</code>). By default, when you hit build/run the assembler will only look for labels in the current file, so if you try and branch <code>bl</code> to one of these functions from <code>main.S</code> it’ll complain that the label doesn’t exist. By marking these functions as <code>.global</code>, it means that the assembler will look everywhere for them, even if they’re in a different source file to the one they’re being called from. Finally, the <code>.global</code> labels in a file are good clue about which functions are useful to call from your own code.</li>

 <li>You may notice things like <code>ldr r0, =ADR_RCC</code> and wonder why it’s not a number after <code>=</code>. <code>ADR_RCC</code> is a symbol declared in <code>lib/util.S</code> using <code>.set</code> directive. This is a somewhat convenient way of abstracting over numeric values.</li>

</ul>

<p class="talk-box">Discuss with your neighbour—what are the advantages of using the functions in the LED library? Are there any disadvantages?

Your task in exercise 1 is to use the functions from the <code>led.s</code> library to write three new <strong>functions</strong> in your <code>main.S</code> file:

<ol>

 <li><code>blink_dot</code>, which blinks the led for a short period of time (say <code>0x20000</code> cycles—we’ll call this the “dot length”) and then pauses (delays) for one dot length before returning</li>

 <li><code>blink_dash</code>, which blinks the led for <em>three</em> times the dot and then pauses (delays) for one dot length before returning</li>

 <li><code>blink_space</code>, which doesn’t blink the LED, but pauses (delays) for <em>seven</em> dot lengths before returning</li>

</ol>

Each of these function calls will contain nested function calls (i.e. calls to <code>delay</code> or other functions) so make sure you use the stack to preserve the link and argument registers (e.g. with <code>push</code> and <code>pop</code>) when necessary.

<p class="push-box">Once you’ve written those functions, write a <code>main</code> loop which blinks out the sequence <code>... _ _ _ </code>on an endless repeat. Commit &amp; push your program to GitLab.

<h2 id="exercise-2-a-morse-data-structure">Exercise 2: a morse data structure</h2>

Now it’s time for the actual morse code part. In morse code, each letter (also called a <strong>codepoint</strong>) is encoded using <em>up to</em> five dots/dashes. For example, the codepoint for the letter B has 4 dots/dashes: <code>_...</code> while the codepoint for the letter E is just a single dot <code>.</code>. You could store this in memory in several different ways, but one way to do it is to use a data structure which looks like this:

<img decoding="async" alt="Morse data structure" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-8/morse-data-structure.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-8/morse-data-structure.png?w=980&amp;ssl=1" alt="Morse data structure" data-recalc-dims="1">

 </noscript>

Each “slot” in the data structure is one full word (32 bits/4 bytes), so the total size of the codepoint data structure is 4*6=24 bytes. The first word is an integer which gives the total number of dots/dashes in the codepoint, while the remaining 5 boxes contain either a 0 (for a dot) or a 1 (for a dash).

<p class="think-box">What will the address offsets for the different slots be? Remember that each box is one 32-bit word in size, but that memory addresses go up in <strong>bytes</strong> (8 bits = 1 byte).

Here are a couple of examples… codepoint B (<code>_...</code>):

<img decoding="async" alt="Morse data B" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-8/morse-data-structure-B.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-8/morse-data-structure-B.png?w=980&amp;ssl=1" alt="Morse data B" data-recalc-dims="1">

 </noscript>

and codepoint E (<code>.</code>)

<img decoding="async" alt="Morse data E" data-recalc-dims="1" data-src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-8/morse-data-structure-E.png?w=980&amp;ssl=1" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==">

 <noscript>

  <img decoding="async" src="https://i0.wp.com/cs.anu.edu.au/courses/comp2300/assets/labs/lab-8/morse-data-structure-E.png?w=980&amp;ssl=1" alt="Morse data E" data-recalc-dims="1">

 </noscript>

In each case, the “end” slots in the data structure might be unused, e.g. if the codepoint only has 2 dots/dashes then the final 3 slots will be unused, and it doesn’t matter if they’re 0 or 1. These slots are coloured a darker grey in the diagrams. (If this inefficiency bums you out, you’ll get a chance to fix it in <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/08-data-structures/#exercise-4">Exercise 4</a>).

Your job Exercise 1 is to write a function which is passed (as a parameter) the base address (i.e. the address of the first slot) of one of these morse data structures and “blinks out” the codepoint using the LED.

As a hint, here are the steps to follow:

<ol>

 <li>pick any character from the morse code table at the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/08-data-structures/#introduction">start of this lab content</a></li>

 <li>store that character in memory (i.e. use the <code>.data</code> section) using the morse codepoint data structure shown in the pictures above</li>

 <li>write a <code>blink_codepoint</code> function which:

  <ul>

   <li>takes the base address of the data structure as an argument in <code>r0</code></li>

   <li>reads the “size” of the codepoint from the first slot</li>

   <li>using that size information, loops over the other slots to blink out the dots/dashes for that codepoint (use the <code>blink_dot</code> and <code>blink_dash</code> functions you wrote earlier)</li>

   <li>when it’s finished all the dots/dashes for the codepoint, delays for 3x dot length (the gap between characters)</li>

  </ul></li>

</ol>

Since the <code>blink_codepoint</code> function will call a bunch of other functions, make sure you use the stack to keep track of values you care about. If your program’s not working properly, make sure you’re not relying in something staying in <code>r0</code> between function calls!

<p class="think-box">When you start to use functions, the usefulness of the <strong>step over</strong> vs <strong>step in</strong> buttons in the debugger toolbar starts to become clear. When the debugger is paused at a function call (i.e. a <code>bl</code> instruction) then step <strong>over</strong> will branch, do the things without pausing, and then pause when the function <em>returns</em>, while step <strong>in</strong> will follow the branch, allowing you to step through the called function as well. Sometimes you want to do one, sometimes you want to do the other, so it’s useful to have both and to choose the right one for the job.

<p class="push-box">Write a program which uses the morse data structure and your <code>blink_codepoint</code> function to blink out the first character of your name on infinite repeat. Commit &amp; push your program to GitLab.

<h2 id="exercise-3-ascii-to-morse-conversion">Exercise 3: <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/ASCII">ASCII</a> to morse conversion</h2>

The final part of today’s lab is to bring it all together to write a program which takes an input string (i.e. a sequence of <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/ASCII">ASCII</a> characters) and blinks out the morse code for that string.

To save you the trouble of writing out the full morse code alphabet, you can copy-paste the following code into your editor. It also includes a place to put the input string (using the <code>.asciz</code> directive).

<pre><code class="language-ARM hljs"><span class="hljs-symbol">.data</span><span class="hljs-symbol">input_string</span>:<span class="hljs-symbol">.asciz</span> <span class="hljs-string">"INPUT STRING"</span><span class="hljs-comment">@ to make sure our table starts on a word boundary</span><span class="hljs-symbol">.align</span> <span class="hljs-number">2</span><span class="hljs-comment">@ Each entry in the table is 6 words long</span><span class="hljs-comment">@ - The first word is the number of dots and dashes for this entry</span><span class="hljs-comment">@ - The next 5 words are 0 for a dot, 1 for a dash, or padding (value doesn't matter)</span><span class="hljs-comment">@</span><span class="hljs-comment">@ E.g., 'G' is dash-dash-dot. There are 2 extra words to pad the entry size to 6 words</span><span class="hljs-symbol">morse_table</span>:  <span class="hljs-meta">.word</span> <span class="hljs-number">2</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ A</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ B</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ C</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">3</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ D</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ E</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ F</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">3</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ G</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ H</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">2</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ I</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ J</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">3</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ K</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ L</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">2</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ M</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">2</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ N</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">3</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ O</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ P</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ Q</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">3</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ R</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">3</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ S</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ T</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">3</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ U</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ V</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">3</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ W</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ X</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ Y</span>  <span class="hljs-meta">.word</span> <span class="hljs-number">4</span>, <span class="hljs-number">1</span>, <span class="hljs-number">1</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span>, <span class="hljs-number">0</span> <span class="hljs-comment">@ Z</span></code></pre>

The main addition you’ll need to make to your program to complete this exercise is a <code>morse_table_index</code> function which takes a single <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/ASCII">ASCII</a> character as input, and returns the base address of the corresponding codepoint data structure for that character (which you can then pass to your <code>blink_codepoint</code> function). For example, the letter P is <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/ASCII">ASCII</a> code <code>80</code>, and the offset of the P codepoint data structure in the table above is 15 (P is the 16th letter) times 24 (size of each codepoint data structure) equals 360 bytes.

So, your main program must:

<ol>

 <li>loop over the characters in the input string (<code>ldrb</code> will be useful here)</li>

 <li>if the character is <code>0</code>, you’re done</li>

 <li>if the character is not <code>0</code>:

  <ul>

   <li>calculate the address of the morse data structure for that character</li>

   <li>call the <code>blink_codepoint</code> function with that base address to blink out the character</li>

   <li>jump back to the top of the loop and repeat for the next character</li>

  </ul></li>

</ol>

If you like, you can modify your program so that any non-capital letter (i.e. <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/ASCII">ASCII</a> value not between 65 and 90 inclusive) will get treated as a space (<code>blink_space</code>).

<p class="push-box">Write a program which blinks out <strong>your name</strong> in morse code. Commit &amp; push this program to GitLab.

<h2 id="exercise-4">Exercise 4: choose-your-own-adventure</h2>

<h3 id="summary">Summary</h3>

5/5 - (1 vote)

<iframe frameborder="0" allowfullscreen data-mce-fragment="1" data-src="https://www.youtube.com/embed/wL2ED9-pACs" class="lazyload" src="data:image/gif;base64,R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw=="></iframe>

There are many ways you can extend this program. Here are a few things to try (pick which ones interest you—you don’t have to do them in order):

<ol>

 <li>can you modify your program to accept both lowercase and uppercase <a class="acton-tabs-link-processed" href="https://en.wikipedia.org/wiki/ASCII">ASCII</a> input?</li>

 <li>the current <code>morse_table</code> doesn’t include the numbers 0 to 9; can you modify your program to handle these as well?</li>

 <li>can you remove the need for the number of dots/dashes in each table entry altogether?</li>

 <li>this is <strong>far</strong> from the most space-efficient way to store the morse codepoints, can you implement a better scheme?</li>

 <li>can you modify the <code>led.S</code> library to set up and blink the <a class="acton-tabs-link-processed" href="https://cs.anu.edu.au/courses/comp2300/labs/06-blinky/#exercise-2">green LED</a> as well—how can you use this in your morse blinking?</li>