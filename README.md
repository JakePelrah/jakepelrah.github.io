<!doctype html>
<html lang='en'>
<head>
    <style>body{ margin:0; background:black; }</style>
    <script src="https://cdn.jsdelivr.net/npm/tweakpane@3.0.7/dist/tweakpane.min.js"></script>
 </head>
<body>
<canvas id='gl'></canvas>
</body>
<!-- vertex shader, as simple as possible -->
<script id='vertex' type='x-shader/x-vertex'>
    attribute vec2 a_position;

    void main() {
      gl_Position = vec4( a_position, 0, 1 );
    }
  </script>

<!-- game of life simulation fragment shader -->
<script id='simulation' type='x-shader/x-fragment'>
   #ifdef GL_ES
precision mediump float;
#endif

uniform float time;
uniform vec2 resolution;

uniform float cloudB;
uniform float cloudG;
uniform float cloudK;
uniform float cloudF;

uniform float treeB;
uniform float treeG;
uniform float treeK;
uniform float treeF;

uniform float rootsB;
uniform float rootsG;
uniform float rootsK;
uniform float rootsF;


// simulation texture state, swapped each frame
uniform sampler2D state;


// look up individual cell values
vec3 get(int x, int y) {
  return texture2D( state, ( gl_FragCoord.xy + vec2(x, y) ) / resolution ).rgb;
}

void main() {

    float dB, dG, f, k;
    vec3 laplace = get(0, 0).rgb * -1.0 +
            get(-1,  0).rgb * 0.2 +
            get( 1,  0).rgb * 0.2 +
            get( 0, -1).rgb * 0.2 +
            get( 0,  1).rgb * 0.2 +
            get(-1, -1).rgb * 0.05 +
            get(-1,  1).rgb * 0.05 +
            get( 1, -1).rgb * 0.05 +
            get( 1,  1).rgb * 0.05;


    vec3 rgb = texture2D(state, gl_FragCoord.xy / resolution ).rgb;
    vec2 st = gl_FragCoord.xy / resolution;


    //Clouds
    if(st.y > 0.75){
        dB = cloudB;
        dG = cloudG;
        f = cloudF;
        k = cloudK;
    }

    // Tree
    else if(st.y > 0.5){
        dB = treeB;
        dG = treeG;
        f = treeF;
        k = treeK;
    }

    // Roots
    else if(st.y < 0.55){
        dB = rootsB;
        dG = rootsG;
        f = rootsF;
        k = rootsK;
    }

     rgb.b += (dB * laplace.b) - (rgb.b * rgb.g *rgb.g) + f * (1.0 - rgb.b);
     rgb.g += (dG * laplace.g) + (rgb.b * rgb.g * rgb.g) - (k + f) * rgb.g;

     gl_FragColor = vec4(0.0, rgb.g, rgb.b, 1.0);

}
  </script>

<!-- render to screen shader -->
<script id='render' type='x-shader/x-fragment'>
    #ifdef GL_ES
    precision mediump float;
    #endif

    uniform sampler2D uSampler;
    uniform vec2 resolution;

    void main() {
    vec2 st = gl_FragCoord.xy / resolution;
    vec3 tex = texture2D( uSampler, gl_FragCoord.xy / resolution ).rgb;

     //Clouds
    if(st.y > 0.75){

      gl_FragColor = vec4(tex.r, tex.g, 1.0, 1.0);
    }

    // Tree
    else if(st.y > 0.5){

      gl_FragColor = vec4(0.2, clamp(tex.g, 0.1, 0.6), 0.0, 1.0);
    }

    // Roots
    else if(st.y < 0.55){
      gl_FragColor = vec4(0.0, tex.g, 0.0, 1.0);
    }

    }
  </script>

<script type='text/javascript'>
    let gl, framebuffer,
        simulationProgram, drawProgram,
        uTime, uSimulationState,
        textureBack, textureFront,
        dimensions = { width:null, height:null }

    window.onload = function() {
        const canvas = document.getElementById( 'gl' )
        gl = canvas.getContext( 'webgl' )
        canvas.width = dimensions.width = window.innerWidth
        canvas.height = dimensions.height = window.innerHeight

        // define drawing area of webgl canvas. bottom corner, width / height
        // XXX can't remember why we need the *2!
        gl.viewport( 0,0, gl.drawingBufferWidth, gl.drawingBufferHeight )


        makeBuffer()
        makeShaders()
        makeTextures()
        setInitialState()


        gl.useProgram(simulationProgram);

        //Clouds
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "cloudB"), 1.0);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "cloudG"), 0.46);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "cloudF"), 0.0545);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "cloudK"), 0.0545);

        //Tree
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "treeB"), 1.0);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "treeG"), 0.4);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "treeF"), 0.0545);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "treeK"), 0.062);

        // Roots
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "rootsB"), 1.25);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "rootsG"), 0.1);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "rootsF"), 0.0367);
        gl.uniform1f(gl.getUniformLocation(simulationProgram, "rootsK"), 0.0649);


        const pane = new Tweakpane.Pane();

        const tab = pane.addTab({
            pages: [
                {title: 'Clouds'},
                {title: 'Tree'},
                {title: 'Roots'}
            ],
        });

        const CloudParams = {
            A: 1.0,
            B: 0.46,
            Feed: 0.0545,
            Kill: 0.0545
        };

       const cloudA =  tab.pages[0].addInput(CloudParams, 'A', {min:0.1, max:1.0, step:.001});
       const cloudB =  tab.pages[0].addInput(CloudParams, 'B', {min:0.1, max:1.0, step:.001});
       const cloudF =  tab.pages[0].addInput(CloudParams, 'Feed', {min:0.01, max:.1, step:.001});
       const cloudK =  tab.pages[0].addInput(CloudParams, 'Kill', {min:0.01, max:.1, step:.001});

        cloudA.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "cloudB"), ev.value);
        })
        cloudB.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "cloudG"), ev.value);
        })

        cloudF.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "cloudF"), ev.value);
        })

        cloudK.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "cloudK"), ev.value);
        })

        const TreeParams = {
            A: 1.0,
            B: 0.4,
            Feed: 0.0545,
            Kill: 0.062
        };

        const treeA =  tab.pages[1].addInput(TreeParams, 'A', {min:0.1, max:1.0, step:.001});
        const treeB =  tab.pages[1].addInput(TreeParams, 'B', {min:0.1, max:1.0, step:.001});
        const treeF =  tab.pages[1].addInput(TreeParams, 'Feed', {min:0.01, max:.1, step:.001});
        const treeK =  tab.pages[1].addInput(TreeParams, 'Kill', {min:0.01, max:.1, step:.001});

        treeA.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "treeB"), ev.value);
        })
        treeB.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "treeG"), ev.value);
        })

        treeF.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "treeF"), ev.value);
        })

        treeK.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "treeK"), ev.value);
        })


        const RootsParams = {
            A: 1.25,
            B: 0.1,
            Feed: 0.0367,
            Kill: 0.0649
        };

        const rootsA =  tab.pages[2].addInput(RootsParams, 'A', {min:0.1, max:2.0, step:.001});
        const rootsB =  tab.pages[2].addInput(RootsParams, 'B', {min:0.1, max:2.0, step:.001});
        const rootsF =  tab.pages[2].addInput(RootsParams, 'Feed', {min:0.01, max:.1, step:.001});
        const rootsK =  tab.pages[2].addInput(RootsParams, 'Kill', {min:0.01, max:.1, step:.001});

        rootsA.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "rootsB"), ev.value);
        })

        rootsB.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "rootsG"), ev.value);
        })

        rootsF.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "rootsF"), ev.value);
        })

        rootsK.on("change", (ev)=>{
            gl.useProgram(simulationProgram);
            gl.uniform1f(gl.getUniformLocation(simulationProgram, "rootsK"), ev.value);
        })

    }

    function makeBuffer() {
        // create a buffer object to store vertices
        const buffer = gl.createBuffer()

        // point buffer at graphic context's ARRAY_BUFFER
        gl.bindBuffer( gl.ARRAY_BUFFER, buffer )

        const triangles = new Float32Array([
            -1, -1,
            1, -1,
            -1,  1,
            -1,  1,
            1, -1,
            1,  1
        ])

        // initialize memory for buffer and populate it. Give
        // open gl hint contents will not change dynamically.
        gl.bufferData( gl.ARRAY_BUFFER, triangles, gl.STATIC_DRAW )
    }

    function makeShaders() {
        // create vertex shader
        let shaderScript = document.getElementById('vertex')
        let shaderSource = shaderScript.text
        const vertexShader = gl.createShader( gl.VERTEX_SHADER )
        gl.shaderSource( vertexShader, shaderSource )
        gl.compileShader( vertexShader )

        // create fragment shader
        shaderScript = document.getElementById('render')
        shaderSource = shaderScript.text
        const drawFragmentShader = gl.createShader( gl.FRAGMENT_SHADER )
        gl.shaderSource( drawFragmentShader, shaderSource )
        gl.compileShader( drawFragmentShader )
        console.log( gl.getShaderInfoLog(drawFragmentShader) )

        // create render program that draws to screen
        drawProgram = gl.createProgram()
        gl.attachShader( drawProgram, vertexShader )
        gl.attachShader( drawProgram, drawFragmentShader )

        gl.linkProgram( drawProgram )
        gl.useProgram( drawProgram )

        uRes = gl.getUniformLocation( drawProgram, 'resolution' )
        gl.uniform2f( uRes, gl.drawingBufferWidth, gl.drawingBufferHeight )

        // get position attribute location in shader
        let position = gl.getAttribLocation( drawProgram, 'a_position' )
        // enable the attribute
        gl.enableVertexAttribArray( position )
        // this will point to the vertices in the last bound array buffer.
        // In this example, we only use one array buffer, where we're storing
        // our vertices
        gl.vertexAttribPointer( position, 2, gl.FLOAT, false, 0,0 )

        shaderScript = document.getElementById('simulation')
        shaderSource = shaderScript.text
        const simulationFragmentShader = gl.createShader( gl.FRAGMENT_SHADER )
        gl.shaderSource( simulationFragmentShader, shaderSource )
        gl.compileShader( simulationFragmentShader )
        console.log( gl.getShaderInfoLog( simulationFragmentShader ) )

        // create simulation program
        simulationProgram = gl.createProgram()
        gl.attachShader( simulationProgram, vertexShader )
        gl.attachShader( simulationProgram, simulationFragmentShader )

        gl.linkProgram( simulationProgram )
        gl.useProgram( simulationProgram )

        uRes = gl.getUniformLocation( simulationProgram, 'resolution' )
        gl.uniform2f( uRes, gl.drawingBufferWidth, gl.drawingBufferHeight )

        // find a pointer to the uniform "time" in our fragment shader
        uTime = gl.getUniformLocation( simulationProgram, 'time' )

        //uSimulationState = gl.getUniformLocation( simulationProgram, 'state' )

        position = gl.getAttribLocation( simulationProgram, 'a_position' )
        gl.enableVertexAttribArray( simulationProgram )
        gl.vertexAttribPointer( position, 2, gl.FLOAT, false, 0,0 )
    }

    function makeTextures() {
        textureBack = gl.createTexture()
        gl.bindTexture( gl.TEXTURE_2D, textureBack )

        // these two lines are needed for non-power-of-2 textures
        gl.texParameteri( gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE )
        gl.texParameteri( gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE )

        // how to map when texture element is less than one pixel
        // use gl.NEAREST to avoid linear interpolation
        gl.texParameteri( gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST )
        // how to map when texture element is more than one pixel
        gl.texParameteri( gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST)

        // specify texture format, see https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/texImage2D
        gl.texImage2D( gl.TEXTURE_2D, 0, gl.RGBA, dimensions.width, dimensions.height, 0, gl.RGBA, gl.UNSIGNED_BYTE, null )

        textureFront = gl.createTexture()
        gl.bindTexture( gl.TEXTURE_2D, textureFront )
        gl.texParameteri( gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE )
        gl.texParameteri( gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE )
        gl.texParameteri( gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.NEAREST )
        gl.texParameteri( gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.NEAREST )
        gl.texImage2D( gl.TEXTURE_2D, 0, gl.RGBA, dimensions.width, dimensions.height, 0, gl.RGBA, gl.UNSIGNED_BYTE, null )

        // Create a framebuffer and attach the texture.
        framebuffer = gl.createFramebuffer()

        // textures loaded, now ready to render
        render()
    }

    // keep track of time via incremental frame counter
    let time = 0
    function render() {
        // schedules render to be called the next time the video card requests
        // a frame of video
        window.requestAnimationFrame( render )

        // use our simulation shader
        gl.useProgram( simulationProgram )
        // update time on CPU and GPU
        time++
        gl.uniform1f( uTime, time )
        gl.bindFramebuffer( gl.FRAMEBUFFER, framebuffer )
        // use the framebuffer to write to our texFront texture
        gl.framebufferTexture2D( gl.FRAMEBUFFER, gl.COLOR_ATTACHMENT0, gl.TEXTURE_2D, textureFront, 0 )
        // set viewport to be the size of our state (game of life simulation)
        // here, this represents the size that will be drawn onto our texture
        gl.viewport(0, 0, dimensions.width,dimensions.height )

        // in our shaders, read from texBack, which is where we poked to
        gl.activeTexture( gl.TEXTURE0 )
        gl.bindTexture( gl.TEXTURE_2D, textureBack )
        gl.uniform1i( uSimulationState, 0 )
        // run shader
        gl.drawArrays( gl.TRIANGLES, 0, 6 )

        // swap our front and back textures
        let tmp = textureFront
        textureFront = textureBack
        textureBack = tmp

        // use the default framebuffer object by passing null
        gl.bindFramebuffer( gl.FRAMEBUFFER, null )
        // set our viewport to be the size of our canvas
        // so that it will fill it entirely
        gl.viewport(0, 0, dimensions.width,dimensions.height )
        // select the texture we would like to draw to the screen.
        // note that webgl does not allow you to write to / read from the
        // same texture in a single render pass. Because of the swap, we're
        // displaying the state of our simulation ****before**** this render pass (frame)
        gl.bindTexture( gl.TEXTURE_2D, textureFront )
        // use our drawing (copy) shader
        gl.useProgram( drawProgram )
        // put simulation on screen
        gl.drawArrays( gl.TRIANGLES, 0, 6 )
    }

    function poke( x, y, color, texture ) {
        gl.bindTexture( gl.TEXTURE_2D, texture )

        // https://developer.mozilla.org/en-US/docs/Web/API/WebGLRenderingContext/texSubImage2D
        gl.texSubImage2D(
            gl.TEXTURE_2D, 0,
            // x offset, y offset, width, height
            x, y, 1, 1,
            gl.RGBA, gl.UNSIGNED_BYTE,
            // is supposed to be a typed array
            new Uint8Array([ ...color, 255 ])
        )
    }

    // set all to white
    function setInitialState() {
        let w = dimensions.width
        let h = dimensions.height

        // Seed canvas
        for(let i = 0; i < dimensions.width; i++ ) {
            for(let j = 0; j < dimensions.height; j++ ) {
                    poke( i, j, [0,0,255], textureBack )
            }
        }


        // Clouds
        for(let i = 0; i < dimensions.width; i++ ) {
            for (let j = 0; j < dimensions.height; j++) {
                if (j > 600 && Math.random() < 0.005) {
                    poke(i, j, [0, 255, 0], textureBack)
                }
            }
        }


        // Tree
        for(let i = w/2 - 20; i < w/2 + 20; i++ ) {
            for (let j = h/2 - 20; j < h/2 +20; j++) {
                poke(i, j, [0, 255, 0], textureBack)
            }
        }



        // Roots
        for(let i = 0; i < dimensions.width; i++ ) {
            for(let j = 0; j < dimensions.height; j++ ) {
                if (j < (h / 2) && j > (h/2 - 10) ) {
                    poke(i, j, [0, 255, 0], textureBack)
                }
            }
        }




    }


</script>

</html>
