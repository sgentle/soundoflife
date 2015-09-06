The Sound of Life
=================

The Sound of Life is an audio automata playground built on [Conway's Game of
Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life) using a
combination of Canvas and Web Audio.

You can view a demo here: http://sgentle.github.io/soundoflife


Setup
-----

Some standard macguffins for dealing with the DOM and setting up our canvas.

    $ = document.querySelector.bind(document)
    EventTarget.prototype.on = EventTarget.prototype.addEventListener #YOLO
    canvas = $('#canvas')
    ctx = canvas.getContext("2d")
    ctx.fillRect(0, 0, canvas.width, canvas.height)


Constants. Uh, "constants". We can change these from the UI because, like,
change is the only constant, man.

    TONES = 5 # Also segment width
    OCTAVES = 4 # Also segment height

    MID_NOTE = 440

    SCALE_X = 4 # GoL cells per tone
    SCALE_Y = 5 # GoL cells per octave

    TIME_SCALE = 5 # Draws per step


This is the best way I can think of to put these in the top scope while still
being able to re-derive them when I need to.

    data = newdata = drawdata = DATA_W = DATA_H = DATA_LENGTH = null #Eww


We re-run this whenever the constants change. The two data buffers run the GoL
itself, holding the current and previous step respectively. We swap them a
lot.

    setupBuffers = ->
      DATA_W = SCALE_X * TONES
      DATA_H = SCALE_Y * OCTAVES

      DATA_LENGTH = DATA_W*DATA_H

      data = new Uint8ClampedArray(DATA_LENGTH)
      newdata = new Uint8ClampedArray(DATA_LENGTH)

      drawdata = new Uint8ClampedArray(DATA_LENGTH*4) #RGBA

      for i in [0...data.length]
        data[i] = newdata[i] = 0
      for i in [0...drawdata.length]
        drawdata[i] = if i % 4 is 3 then 255 #Alpha channel


Regions
-------

We divide the grid into equally-sized regions. Each region has an oscillator,
a gain node and a color. We set the frequency here and then vary the gain with
the sum of the active cells in that region. Similarly, we set the hue and
saturation of the cells here and then set the value based on whether the
cell's alive.

    genColor = (h, s) ->
      i = Math.floor(h * 6)
      f = h * 6 - i
      p = 1 - s
      q = 1 - f * s
      t = 1 - (1 - f) * s
      v = 1
      [r, g, b] =
        switch i % 6
          when 0 then [v, t, p]
          when 1 then [q, v, p]
          when 2 then [p, v, t]
          when 3 then [p, q, v]
          when 4 then [t, p, v]
          when 5 then [v, p, q]

      {r, g, b}

Everything connects to the master gain in case we want to add a volume control
or something. I set it to 0.75 even though the correct value is 1 because
people aren't used to the raw power of properly normalised audio.

    context = mastergain = null
    setupAudio = ->
      context = new (AudioContext or webkitAudioContext)()
      mastergain = context.createGain()
      mastergain.gain.value = 0.75
      mastergain.connect context.destination

Set up our regions. The heavy lifting for the colors is done already, but we
still have to set up the frequencies. The magic formula below creates an
N-tone equal temperament scale centered on MID_NOTE which is A440 because it's
the only frequency I can remember.

    regions = []

    setupRegions = ->
      region.osc.stop() for region in regions

      regions = []
      for i in [0...TONES * OCTAVES]
        freq = MID_NOTE * Math.pow(2, i/TONES - OCTAVES/2)

        osc = context.createOscillator()
        osc.frequency.value = freq

        gain = context.createGain()
        gain.gain.value = 0

        osc.connect gain
        osc.start()

        gain.connect mastergain

        x = i % TONES / TONES
        y = i // TONES / OCTAVES
        color = genColor x, 1-y

        regions.push {osc, gain, color, newgain: 0}


Turn grid coordinates into regions. Slight quirk: regions are numbered from
the bottom left, grids are from the top left. Canvas likes the latter, but it
seems more natural for pitch to go up as it goes... up.

    getRegion = (x, y) ->
      y = (DATA_H - 1 - y) // SCALE_Y
      x //= SCALE_X
      reg = regions[x + y*TONES]


Drawing
-------

To draw fast and feel like super cool hackers, we just render the pixels
directly onto the canvas and then scale them up. To do this - I'm not even
kidding - we draw the data to the canvas, then draw the canvas into itself.

    setupCanvas = ->
      ctx.setTransform(canvas.width/DATA_W, 0, 0, canvas.height/DATA_H, 0, 0)
      ctx.globalCompositeOperation = "copy"
      ctx.imageSmoothingEnabled = false
      ctx[x+"ImageSmoothingEnabled"] = false for x in 'moz ms webkit'.split(' ') # Vendor prefixes are gross


Because of the magic of cinema, we want to draw as quickly as we can even
though the GoL can look nicer at a slow speed. To bridge this gap we use
interpolation, which I'm told is latin for "making it look blurry".

We step the GoL every TIME_SCALE draws, and drawcount reflects how many draws
we've done.

    drawcount = 0
    draw = ->
      totalgain = 0

      drawR = drawcount/TIME_SCALE
      for i in [0...drawdata.length] by 4
        region = getRegion(i/4 % DATA_W, i/4 // DATA_W)

        val = data[i>>2]
        oldval = newdata[i>>2]
        v = 255 * (val*drawR + oldval*(1-drawR))

        region.newgain += v
        totalgain += v

        c = region.color
        drawdata[i] = c.r * v
        drawdata[i+1] = c.g * v
        drawdata[i+2] = c.b * v

      max_region = SCALE_X * SCALE_Y
      max_global = DATA_W * DATA_H
      for region in regions
        region.gain.gain.value = region.newgain / totalgain or 0
        region.newgain = 0

      imageData = new ImageData(drawdata, DATA_W, DATA_H)
      ctx.putImageData(imageData, 0, 0)
      ctx.drawImage(ctx.canvas, 0, 0)


Game of Life
------------

Here's the code that actually does things. Everything else is really just
window dressing. Rel gets us the value of each neighbour, or 0 if we go past
the boundary of the screen.

    rel = (n, x, y) ->
      i = n + x + y * DATA_W
      b = n % (DATA_W) + x
      return 0 if b > (DATA_W) or b < 0 or i >= DATA_LENGTH or i < 0 # prevent overflows
      data[i]


Step implements the GoL logic itself. You know I saw someone do this in one
line of APL once? I bet it ran faster too...

    step = ->
      for i in [0...data.length] by 1
        neighbours = [
          rel i, -1, -1 #top left
          rel i,  0, -1 #top
          rel i, +1, -1 #top right

          rel i, -1,  0 #left
          rel i, +1,  0 #right

          rel i, -1, +1 #bottom left
          rel i,  0, +1 #bottom
          rel i, +1, +1 #bottom right
        ]
        alive = 0
        for n in neighbours
          alive++ if n == 1

        if alive < 2
          newdata[i] = 0
        else if alive > 3
          newdata[i] = 0
        else if alive == 3
          newdata[i] = 1
        else
          newdata[i] = data[i]

      [data, newdata] = [newdata, data]


rAF
---

Actually call our code on the graphics loop. Logic as described above in the
draw section.

    paused = false
    raf = ->
      drawcount++ unless paused
      draw()
      if drawcount >= TIME_SCALE
        drawcount = 0
        step()

      requestAnimationFrame raf

Init
----

These are how we start things running. We do this as late as possible so I can
embed this on my website without loading a billion oscillators every time.

    inited = false
    init = ->
      inited = true
      setupAudio()
      setupBuffers()
      setupRegions()
      setupCanvas()
      step()
      raf()

    reInit = ->
      setupBuffers()
      setupRegions()
      setupCanvas()


DOMfoolery
----------

    clicking = false
    click = (ev) ->
      return unless clicking
      x = Math.floor(ev.offsetX / canvas.offsetWidth * DATA_W)
      y = Math.floor(ev.offsetY / canvas.offsetHeight * DATA_H)
      i = y * DATA_W + x
      data[i] = newdata[i] = 1

    canvas.on 'mousedown', (ev) ->
      init() unless inited
      clicking = true if ev.button is 0
      click(ev)
    canvas.on 'mouseup', -> clicking = false
    canvas.on 'mouseout', -> clicking = false
    canvas.on 'mousemove', click

    $('#clear').on 'click', ->
      data[i] = newdata[i] = 0 for i in [0...data.length]
    $('#random').on 'click', ->
      data[i] = newdata[i] = Math.round(Math.random()) for i in [0...data.length]
    $('#pause').on 'click', ->
      paused = !paused
      $('#pause').textContent = if paused then 'Play' else 'Pause'

    $('#octaves').value = OCTAVES
    $('#octaves').on 'input', -> OCTAVES = $('#octaves').value * 1 or OCTAVES; reInit()

    $('#tones').value = TONES
    $('#tones').on 'input', -> TONES = $('#tones').value * 1 or TONES; reInit()

    $('#scale_x').value = SCALE_X
    $('#scale_x').on 'input', -> SCALE_X = $('#scale_x').value * 1 or SCALE_X; reInit()

    $('#scale_y').value = SCALE_Y
    $('#scale_y').on 'input', -> SCALE_Y = $('#scale_y').value * 1 or SCALE_Y; reInit()

    $('#mid_note').value = MID_NOTE
    $('#mid_note').on 'input', -> MID_NOTE = $('#mid_note').value * 1 or MID_NOTE; setupRegions()

    $('#time_scale').value = TIME_SCALE
    $('#time_scale').on 'input', -> TIME_SCALE = $('#time_scale').value * 1 or TIME_SCALE
