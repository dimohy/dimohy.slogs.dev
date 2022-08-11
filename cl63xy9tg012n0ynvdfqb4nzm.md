## TypeScript로 canvas에 원 움직이기

TypeScript를 이용해 canvas에 원을 움직이는 예제를 만들어 보겠습니다.

본격적으로 개발하기 위해 Visual Studio Code에 TypeScript 개발 환경을 만들 것입니다.


## 개발 환경 구성

### TypeScript 컴파일러(tsc) 설치
tsc를 가장 쉽게 설치하는 방법은 npm을 이용하는 것입니다.

```
$ npm install -g typescript
```

정상적으로 설치되었는지 버젼을 통해 확인 합니다.

```
$ tsc --version
```

`Hello World` 예제 부터 시작해 봅시다. `HelloWorld` 디렉토리를 만든 후 VS Code를 실행합니다.

```
$ mkdir HelloWorld
$ cd HelloWorld
$ code .
```

`helloworld.ts`를 만든 후 아래의 코드를 추가합니다.

```typescript
let message: string = "Hello World";
console.log(message);
```

통합 터미널 (Ctrl + \`)을 열고 `tsc helloworld.ts`로 컴파일을 하면 `helloworld.js`를 얻을 수 있습니다. `Node.js`를 통해 실행해 볼 수 있습니다.

```
$ node .\helloworld.js
Hello World
```

## 파일 구성

TypeScript를 이용해 `canvas`에 원을 움직이는 코드를 작성하기 위해서는 다음의 세 개의 파일이 필요합니다.

- `tsconfig.json` : TypeScript 컴파일러를 구성
- `index.html` : `canvas`와 원을 추가하는 `button`이 있는 단순한 HTML 페이지
- `main.ts` : 우리가 작성하고 컴파일 할 TypeScript 파일


### tsconfig.json

파일은 다음과 같이 구성합니다.

```json
{
    "compilerOptions": {
        "target": "es5",
        "sourceMap": true,
        "lib": [
            "es5",
            "dom"
        ],
        "noUnusedLocals": true,
        "module": "commonjs"
    }
}
```

위 설정에 대한 자세한 사항은 TypeScript 문서의 [What is a tsconfig.json](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html)를 통해 확인할 수 있습니다.


### index.html

파일은 다음과 같이 구성합니다. 우리가 TypeScript로 `canvas`에 원을 움직이게 하기 위한 간단한 HTML 구성을 할 것입니다.

```html
<html>

<head>
    <title>Canvas Sample</title>
</head>

<body>
    <canvas id="canvas" width="800" height="800"
            style="border: 1px solid black;">
    </canvas>
    <p></p>
    <button>add 10 circles</button>
    <script src="main.js"></script>
</body>

</html>
```


### main.ts

마지막으로 구현해야 할 TypeScript 파일입니다. 우리는 클래스를 이용할 것이므로 다음의 간단한 구성으로 시작합니다.

```typescript
class DrawingApp {

}

new DrawingApp();
```


## 디버깅 환경 구성

웹페이지로 띄울 것이기 때문에 `Live Server`를 이용해서 이를 설정합니다.

`Live Server`를 VS Code 확장에서 설치 합니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658939780634/9KxORQnXI.png align="left")

설치되면 오른쪽 하단에 `Go Live` 버튼이 생성됩니다. 이 버튼을 누르면 현 디렉토리로 웹 서버를 띄우게 됩니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658939888664/w8yx5UfQi.png align="left")


### TypeScript 빌드

터미널에서 `tsc`를 실행합니다.

```
$ tsc
```

`tsc`를 실행하면 `tsconfig.json` 구성에 의해 `main.js` 및 `main.js.map` 파일이 생성됩니다.

만약 TypeScript 소스코드를 변경할 때마다 자동으로 빌드되게 하려면 `tsconfig.json`의 `compilerOptions`에 `"watch": true`를 추가합니다.

### 크롬에서 디버깅

크롬 브라우저에서 디버깅을 하려면 다음의 `launch.json`을 `.vscode` 디렉토리에 추가 합니다.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "type": "pwa-chrome",
            "request": "launch",
            "name": "Launch Chrome against localhost",
            "url": "http://localhost:5500",
            "webRoot": "${workspaceFolder}"
        }
    ]
}
```

이때 url은 `Live Server`의 실행 포트를 입력하도록 합니다.

이제 적절한 위치에 중단점 (F9)을 지정하고 `디버깅 시작 (F5)`를 통해 디버깅을 시작할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658940797393/seowx7rig.png align="left")

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658940810586/tLwHjZV6l.png align="left")


## 샘플 코딩

디버깅 환경이 구성되었으므로 코드를 조금 바꿔 봅시다.

| main.ts
```typescript
class DrawingApp {
    private canvas: HTMLCanvasElement;
    private context: CanvasRenderingContext2D;

    constructor(canvas: HTMLCanvasElement) {
        this.canvas = canvas;
        this.context = canvas.getContext("2d");

        this.context.fillStyle = "blue";
        this.context.beginPath();
        this.context.arc(100, 100, 16, 0, 2 * Math.PI);
        this.context.fill();
    }
}
```

`canvas`의 `context`를 획득해서 청색 원을 `canvas`에 그려보았습니다. `constructor`에 `canvas` 객체를 넘겨줘야 하므로 `index.html`도 다음처럼 변경 합니다.

| index.html
```html
<html>

<head>
    <title>Canvas Sample</title>
</head>

<body>
    <canvas id="canvas" width="800" height="800"
            style="border: 1px solid black;">
    </canvas>
    <p></p>
    <button>add 10 circles</button>
    <script src="main.js"></script>
    <script type="text/javascript">
        const app = new DrawingApp(document.getElementById('canvas'));
    </script>
</body>

</html>
```

실행하면 다음과 같이 `canvas`에 청색 원이 그려지는 것을 확인할 수 있습니다.

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658941998006/vzLxh8bmQ.png align="left")

`Live Server`는 파일을 감시해서 변경되면 웹페이지를 갱신하는 기능이 있어서 `index.html`을 변경하면 웹페이지에 바로 반영을 해줍니다.

또한 `tsconfig.json`의 `compilerOptions`의 `"watch": true` 설정에 의해 `main.ts`가 변경될 때 `tsc`가 `main.js`로 컴파일 해주고 `Live Server`가 이를 감지해 웹페이지에 반영해 줍니다.

| `main.js`의 `this.context.fillStyle = "blue";`를 `this.context.fillStyle = "red";`로 변경한 후 파일 저장시

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658942294319/qKlLD2SIT.png align="left")


### `canvas` 다시 그리기

원을 이동하게 하려면 주기적으로 `canvas`를 다시 그려야 하는데 웹브라우저에서 다시 그릴  수 있는 시점을 알아야 합니다. 이 때 `requestAnimationFrame()`를 이용할 수 있습니다.

```typescript
class DrawingApp {
    private canvas: HTMLCanvasElement;
    private context: CanvasRenderingContext2D;

    constructor(canvas: HTMLCanvasElement) {
        // ...

        requestAnimationFrame(this.redraw);
    }

    private redraw() {
        console.log("!!!");

        requestAnimationFrame(this.redraw);
    }
}
```

그런데 이 코드는 한번 `redraw()`가 호출된 후 다시  `requestAnimationFrame(this.redraw);`가 실행될 때 정상 동작하지 않습니다. 왜냐하면 콜백으로 호출된 메서드에서 `this`는 `DrawingApp` 객체가 아닌 `requestAnimationFrame()` 메서드가 있는 `Window` 객체가 되기 때문입니다.

이를 개선하기 위해 `redraw()`를 다음처럼 변경해봅시다.

```typescript
    private redraw = () => {
        console.log("!!!");

        requestAnimationFrame(this.redraw);
    }
```

이제 잘 `redraw()`가 계속해서 호출이 됩니다. `this`의 작동 방식 및 특징 및 수정 방법은 [`this` in TypeScript](https://github.com/Microsoft/TypeScript/wiki/'this'-in-TypeScript)를 참고하세요.


### 원 구조

원을 표시하려면 원의 좌표 정보 및 반지름, 색 등의 정보를 따로 정의하는 것이 좋습니다.

```typescript
type Circle = {
    x: number,
    y: number,
    readonly radius: number,
    readonly color: string,
    xInc: number,
    yInc: number,
};
```

이제 원 목록을 담는 클래스 필드 및 색 관련 필드를 추가합니다. 

```typescript
    private readonly circles: Circle[] = [];
    private readonly colors: readonly string[] = ["red", "green", "blue"];
    private colorsCount: number = 0;
```


### 원 관련 기능

원을 추가할 때마다 다른 색을 적용하기 위해 `getColor()` 메서드를 만들고

```typescript
    private getColor(): string {
        let result = this.colors[this.colorsCount];
        this.colorsCount = (this.colorsCount + 1) % this.colors.length;
        return result;
    }
```

원을 추가하는 메서드를 만들고,

```typescript
    public addCircles = (numbers: number) => {
        var color = this.getColor();
        for (var i = 0; i < numbers; i++) {
            this.circles.push({
                x: this.canvas.width * Math.random(),
                y: this.canvas.height * Math.random(),
                radius: 32 * Math.random() + 8,
                xInc: 5 * Math.random() + 1,
                yInc: 5 * Math.random() + 1,
                color: color
            });
        }
    }
```

다음은 원을 그리고 이동하는 메서드를 만듭니다.

```typescript
    private drawCircle(circle: Circle) {
        this.context.fillStyle = circle.color;
        this.context.beginPath();
        this.context.arc(circle.x, circle.y, circle.radius, 0, 2 * Math.PI);
        this.context.fill();
    }

    private moveCircle(circle: Circle) {
        circle.x += circle.xInc;
        circle.y += circle.yInc;

        if (circle.x < 0 || circle.x > this.canvas.width) {
            circle.xInc *= -1;
        }
        if (circle.y < 0 || circle.y > this.canvas.height) {
            circle.yInc *= -1;
        }
    }
```

### 전체 소스코드

| main.ts
```typescript
class DrawingApp {
    private canvas: HTMLCanvasElement;
    private context: CanvasRenderingContext2D;

    private readonly circles: Circle[] = [];
    private readonly colors: readonly string[] = ["red", "green", "blue"];
    private colorsCount: number = 0;


    constructor(canvas: HTMLCanvasElement) {
        this.canvas = canvas;
        this.context = canvas.getContext("2d");

        this.addCircles(10);

        requestAnimationFrame(this.redraw);
    }

    private redraw = () => {
        this.context.fillStyle = "white";
        this.context.fillRect(0, 0, this.canvas.width, this.canvas.height);

        for (const circle of this.circles) {
            this.drawCircle(circle);
            this.moveCircle(circle);
        }

        requestAnimationFrame(this.redraw);
    }

    private drawCircle(circle: Circle) {
        this.context.fillStyle = circle.color;
        this.context.beginPath();
        this.context.arc(circle.x, circle.y, circle.radius, 0, 2 * Math.PI);
        this.context.fill();
    }

    private getColor(): string {
        let result = this.colors[this.colorsCount];
        this.colorsCount = (this.colorsCount + 1) % this.colors.length;
        return result;
    }

    private moveCircle(circle: Circle) {
        circle.x += circle.xInc;
        circle.y += circle.yInc;

        if (circle.x < 0 || circle.x > this.canvas.width) {
            circle.xInc *= -1;
        }
        if (circle.y < 0 || circle.y > this.canvas.height) {
            circle.yInc *= -1;
        }
    }

    public addCircles = (numbers: number) => {
        var color = this.getColor();
        for (var i = 0; i < numbers; i++) {
            this.circles.push({
                x: this.canvas.width * Math.random(),
                y: this.canvas.height * Math.random(),
                radius: 32 * Math.random() + 8,
                xInc: 5 * Math.random() + 1,
                yInc: 5 * Math.random() + 1,
                color: color
            });
        }
    }
}

type Circle = {
    x: number,
    y: number,
    readonly radius: number,
    readonly color: string,
    xInc: number,
    yInc: number,
};
```

다음으로 버튼을 눌렀을 때 원을 추가할 수 있도록 `index.html`을 다음과 같이 수정합니다.

| index.html
```html
<html>

<head>
    <title>Canvas Sample</title>
</head>

<body>
    <canvas id="canvas" width="800" height="800"
            style="border: 1px solid black;">
    </canvas>
    <p></p>
    <button id="add">add 10 circles</button>
    <script src="main.js"></script>
    <script type="text/javascript">
        const app = new DrawingApp(document.getElementById('canvas'));

        const button = document.getElementById('add');
        button.onclick = () => app.addCircles(10);
    </script>
</body>

</html>
```

## 실행 결과
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658945799689/QNHHSfY1A.png align="left")


## `tsc`에 의해 변환된 JavaScript 코드

컴파일된 JavaScript 코드를 살펴보는 것은 흥미롭니다. 다음은 컴파일된 JavaScript  코드입니다.

```javascript
var DrawingApp = /** @class */ (function () {
    function DrawingApp(canvas) {
        var _this = this;
        this.circles = [];
        this.colors = ["red", "green", "blue"];
        this.colorsCount = 0;
        this.redraw = function () {
            _this.context.fillStyle = "white";
            _this.context.fillRect(0, 0, _this.canvas.width, _this.canvas.height);
            for (var _i = 0, _a = _this.circles; _i < _a.length; _i++) {
                var circle = _a[_i];
                _this.drawCircle(circle);
                _this.moveCircle(circle);
            }
            requestAnimationFrame(_this.redraw);
        };
        this.addCircles = function (numbers) {
            var color = _this.getColor();
            for (var i = 0; i < numbers; i++) {
                _this.circles.push({
                    x: _this.canvas.width * Math.random(),
                    y: _this.canvas.height * Math.random(),
                    radius: 32 * Math.random() + 8,
                    xInc: 5 * Math.random() + 1,
                    yInc: 5 * Math.random() + 1,
                    color: color
                });
            }
        };
        this.canvas = canvas;
        this.context = canvas.getContext("2d");
        this.addCircles(10);
        requestAnimationFrame(this.redraw);
    }
    DrawingApp.prototype.drawCircle = function (circle) {
        this.context.fillStyle = circle.color;
        this.context.beginPath();
        this.context.arc(circle.x, circle.y, circle.radius, 0, 2 * Math.PI);
        this.context.fill();
    };
    DrawingApp.prototype.getColor = function () {
        var result = this.colors[this.colorsCount];
        this.colorsCount = (this.colorsCount + 1) % this.colors.length;
        return result;
    };
    DrawingApp.prototype.moveCircle = function (circle) {
        circle.x += circle.xInc;
        circle.y += circle.yInc;
        if (circle.x < 0 || circle.x > this.canvas.width) {
            circle.xInc *= -1;
        }
        if (circle.y < 0 || circle.y > this.canvas.height) {
            circle.yInc *= -1;
        }
    };
    return DrawingApp;
}());
//# sourceMappingURL=main.js.map
```


## 정리

VS Code에 TypeScript 개발환경을 구성해서 코딩 결과를 웹브라우저를 통해 확인할 수 있도록 예제를 통해 확인해 보면서 TypeScript의 강력한 타입 검사 기능과 VS Code를 이용한 편리한 디버깅 환경을 살펴볼 수 있었습니다.

## 샘플 데모
https://www.maum.in/samples/moving_circles/index.html
