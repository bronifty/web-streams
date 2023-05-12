# web-streams

- inspiration [vercel ssr streams](https://vercel.com/docs/concepts/functions/edge-functions/streaming)
- education [web.dev/streams](https://web.dev/streams/)

### Operation Types

- Reading
- Writing
- Transforming

### Resource Types

- text
- binary

### ReadableStream

```js
new ReadableStream();
new ReadableStream(underlyingSource);
new ReadableStream(underlyingSource, queuingStrategy);
```

- a readable stream of byte data whose concrete implementation is provided by the Fetch API through the body property of the Response object.

```js
let reader;
const getReader = async () => {
  const res = await fetch("../media/test.png").then(
    (response) => response.body
  );
  reader = res.getReader();
  return reader;
};
getReader();
```

- it is locked to a block of memory and cannot be shared between threads

## Text Resource Examples

- example 1 create a readable stream and read it, pushing the output to the DOM

```html
<html>
  <body>
    <div id="output"></div>
    <script>
      const readableStream = new ReadableStream({
        start(controller) {
          controller.enqueue(new TextEncoder().encode("Hello "));
          setTimeout(() => {
            controller.enqueue(new TextEncoder().encode("world"));
            controller.close();
          }, 1000);
        },
      });

      const reader = readableStream.getReader();

      const push = () => {
        reader.read().then(({ done, value }) => {
          if (done) {
            return;
          }
          document.querySelector("#output").textContent +=
            new TextDecoder().decode(value);
          console.log(new TextDecoder().decode(value));
          push();
        });
      };
      push();
    </script>
    <style>
      body {
        background: #222;
        color: #ddd;
        font-size: 2rem;
      }
    </style>
  </body>
</html>
```

- example 2 create a reusable fetcher

  ```html
  <html>
    <body>
      <div id="output"></div>
      <script>
        // the output element
        const outputElement = document.querySelector("#output");
        // the reader
        let reader;
        // create a reusable fetcher function
        const push = async (reader) => {
          const { done, value } = await reader.read();
          if (done) {
            return;
          }
          outputElement.textContent += new TextDecoder().decode(value);
          push(reader);
        };

        const textFetcher = async (url) => {
          let res = await fetch(url).then((response) => response.body);
          reader = res.getReader();
          push(reader);
        };
        textFetcher("../media/test.txt");
      </script>
      <style>
        body {
          background: #222;
          color: #ddd;
          font-size: 2rem;
        }
      </style>
    </body>
  </html>
  ```

- example 3 in pursuit of a more robust app

```html
<html>
  <body>
    <div id="output"></div>
    <script>
      // input variables
      let url = "../media/test.txt";
      let outputElement = document.querySelector("#output");

      const pushText = async ({ reader, outputElement }) => {
        const { done, value } = await reader.read();
        if (done) {
          return;
        }
        output.textContent += new TextDecoder().decode(value);
        pushText({ reader });
      };
      const getReader = async ({ url }) => {
        const res = await fetch(url).then((response) => response.body);
        const reader = res.getReader();
        return reader;
      };
      const kickoffStream = async ({ url, outputElement }) => {
        const reader = await getReader({ url });
        pushText({ reader, outputElement });
      };
      kickoffStream({ url, output });
      // getReader({ url }).then((reader) => pushText({ reader, outputElement }));
    </script>
    <style>
      body {
        background: #222;
        color: #ddd;
        font-size: 2rem;
      }
    </style>
  </body>
</html>
```

## Binary Resource Examples

- example 5 fetch a binary resource and display it in the DOM

```html
<html>
  <body>
    <h1>index-5!</h1>
    <img
      src="../media/tortoise.png"
      width="150"
      height="84"
      alt="Tortoise Original" />
    <img id="imageOutput" src="" alt="Test Image" />
    <script>
      const imageOutputElement = document.getElementById("imageOutput");

      async function fetchImageAsReadableStreamReader() {
        const res = await fetch("../media/test.png");
        const rb = res.body;
        const reader = rb.getReader();
        return reader;
      }

      async function createStream(reader) {
        return new ReadableStream({
          async start(controller) {
            while (true) {
              const { done, value } = await reader.read();

              if (done) {
                break;
              }

              controller.enqueue(value);
            }

            controller.close();
            reader.releaseLock();
          },
        });
      }

      async function getBlobFromStream(stream) {
        const response = new Response(stream);
        return await response.blob();
      }

      function createObjectURL(blob) {
        return URL.createObjectURL(blob);
      }

      function updateImageSource(url) {
        imageOutputElement.src = url;
      }

      async function fetchAndDisplayImage() {
        try {
          const reader = await fetchImageAsReadableStreamReader();
          const stream = await createStream(reader);
          const blob = await getBlobFromStream(stream);
          const url = createObjectURL(blob);
          updateImageSource(url);
        } catch (error) {
          console.error(error);
        }
      }

      fetchAndDisplayImage();
    </script>
    <style>
      body {
        background: #222;
        color: #ddd;
        font-size: 2rem;
      }
      img {
        width: 200px;
        height: 200px;
      }
    </style>
  </body>
</html>
```

- example-6 a simpler design to showcase fetch body itself returns a ReadableStream; it doesn't need to be explicitly created

```html
<html>
  <body>
    <img
      src="../media/tortoise.png"
      width="150"
      height="84"
      alt="Tortoise Original" />
    <img id="imageOutput" src="" alt="Test Image" />
    <script>
      const imageOutputElement = document.getElementById("imageOutput");

      async function fetchAndDisplayImage() {
        try {
          const response = await fetch("../media/test.png");
          // this part doesn't work
          // const rb = await response.body;
          // const reader = await rb.getReader();
          // const blob = await reader.blob();
          const blob = await response.blob();
          const url = URL.createObjectURL(blob);
          imageOutputElement.src = url;
        } catch (error) {
          console.error(error);
        }
      }

      fetchAndDisplayImage();
    </script>
    <style>
      body {
        background: #222;
        color: #ddd;
        font-size: 2rem;
      }
      img {
        width: 200px;
        height: 200px;
      }
    </style>
  </body>
</html>
```

- example-7 expanding on the simpler design to make each aspect more explicit

```html
<html>
  <body>
    <img
      src="../media/tortoise.png"
      width="150"
      height="84"
      alt="Tortoise Original" />
    <img id="imageOutput" src="" alt="Test Image" />
    <script>
      const imageOutputElement = document.getElementById("imageOutput");

      async function fetchAndDisplayImage() {
        try {
          const response = await fetch("../media/test.png");
          const body = await response.body;
          // fetch response body is an instance of ReadableStream, which has both the reader and data to read
          // first we create a reader from the body
          const reader = await body.getReader();
          // then we create a stream manually to read from the reader
          const createStream = (reader) =>
            new ReadableStream({
              start(controller) {
                function push() {
                  reader.read().then(({ done, value }) => {
                    if (done) {
                      controller.close();
                      return;
                    }
                    controller.enqueue(value);
                    push();
                  });
                }
                push();
              },
            });
          const stream = createStream(reader);
          // now create a blob from the stream
          const blob = await new Response(stream).blob();
          // create an object url from the blob
          const url = URL.createObjectURL(blob);
          imageOutputElement.src = url;
        } catch (error) {
          console.error(error);
        }
      }

      fetchAndDisplayImage();
    </script>
    <style>
      body {
        background: #222;
        color: #ddd;
        font-size: 2rem;
      }
      img {
        width: 200px;
        height: 200px;
      }
    </style>
  </body>
</html>
```
