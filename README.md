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
