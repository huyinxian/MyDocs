# 坦克大战

> 本文仅供个人学习，并不是教程

坦克大战想必各位都玩过，是一款很经典的游戏。这次的目的是做一个小 Demo，实现双人对战的基本功能。

## 导入地图模型

---------
地图模型的名称为`LevelArt`，创建地图之后按照`Windows` -> `Lighting` -> `Settings`打开光照设置：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/01.png)

首先就是要把自动渲染关掉，不然的话编辑器会自动给你渲染，很占资源。然后将`Source`切换为`Color`，根据地图的颜色选择棕黄色，这样地图就看起来没那么亮。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/02.png)

我个人比较喜欢用`Iso`模式，另外摄像机的视角记得调整为`Orthographic`（正交投影）。

## 创建坦克

---------
坦克的创建主要包含三部分：添加特效、碰撞器以及脚本。

### 为坦克添加尘土特效

为了让坦克行驶时显得逼真一些，我们需要为它的两个轮子加上尘土特效。预制体`DustTrail`就是已经做好的尘土。

在`Tank`下创建两个`DustTrail`，分别摆放在轮子下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/03.png)

如果你提前想看一下效果的话，可以给坦克写个控制方向的脚本。

### 为坦克添加碰撞器

点击`Edit Collider`调整碰撞体积，大小的话能够覆盖坦克的主干部分就可以了。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/04.png)

### 编写移动脚本

_TankMovement.cs_

```csharp
public class TankMovement : MonoBehaviour {

	public float speed = 5;            // 坦克前进速度
	public float angularSpeed = 5;     // 坦克旋转速度，度
	public float number = 1;           // 坦克的编号

	private Rigidbody tankBody;

	void Start() {
		tankBody = this.GetComponent<Rigidbody>();
	}

	void FixedUpdate() {
		// Player1 的前进由 W、S 控制，Player2 的前进由 up、down 控制
		float v = Input.GetAxis("VerticalPlayer" + number);

		// Player1 的旋转由 A、D 控制，Player2 的旋转由 left、right 控制
		float h = Input.GetAxis("HorizontalPlayer" + number);

		// 前进、后退
		tankBody.velocity = transform.forward * v * speed;

		// 左右旋转
		tankBody.angularVelocity = transform.up * h * angularSpeed;
	}
}
```

由于有两个玩家，所以不能够直接用`Horizontal`等预设来控制移动。打开`Input`，复制`Horizontal`，将其命名为`HorizontalPlayer1`。玩家1只需要用到 W、S、A、D，所以要把方向键删掉（玩家2控制方向键上下左右）。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/05.png)

依葫芦画瓢，创建好`HorizontalPlayer2`、`VerticalPlayer1`、`VerticalPlayer2`。这样，我们能够根据编号来区分坦克的操作。

## 创建子弹

---------

### 制作预制体

导入子弹模型，然后为它添加胶囊型碰撞器和刚体。碰撞体积可以参照图片上面的来：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/06.png)

最后，将子弹制作预制体。

?> 如果对于组件有什么疑问，请转到——[Capsule Collider](https://docs.unity3d.com/2017.4/Documentation/Manual/class-CapsuleCollider.html)

### 设置炮口位置

光是创建了子弹还没用，我们还需要知道炮口在哪里。在`Tank`下创建一个空对象`FirePosition`，调整它的位置和角度：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/07.png)

## 坦克开火

---------
我们需要为坦克添加两个额外的脚本，分别用于控制坦克的开火以及坦克的生命值。

### 编写开火脚本

_TankAttack.cs_

```csharp
public class TankAttack : MonoBehaviour {

	public GameObject shellPrefab;                     // 子弹
	public KeyCode fireKey = KeyCode.Space;            // 空格键开火
	public float shellSpeed = 15;                      // 子弹速度

	private Transform firePosition;                    // 炮口位置

	void Start() {
		firePosition = transform.Find("FirePosition");
	}

	void Update() {
		// 监听按键
		if (Input.GetKeyDown(fireKey)) {
			GameObject instance = GameObject.Instantiate(shellPrefab, firePosition.position, firePosition.rotation);
			instance.GetComponent<Rigidbody>().velocity = instance.transform.forward * shellSpeed;
		}
	}
}
```

### 编写生命值脚本

_TankHealth.cs_

```csharp
public class TankHealth : MonoBehaviour {

	public int hp = 100;  // 坦克当前血量

	void TakeDamage() {
		if (hp <= 0) { return; }

		// 坦克扣血
		hp -= Random.Range(10, 20);

		// 坦克死亡时爆炸
		if (hp <= 0) {
			// 销毁坦克
			GameObject.Destroy(this.gameObject);
		}
	}
}
```

写完这两个脚本，坦克的逻辑基本上就完成了。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/08.png)

## 添加特效

---------
虽然坦克既能开炮也会受伤，但若是没有特效的话总显得很突兀。为此，我们需要为子弹和坦克添加爆炸效果。

### 为子弹添加爆炸效果

由于子弹在碰撞到任何物体时都会立即发生爆炸，所以我们应该把`Collider`组建中的`Is Trigger`勾选上。接下来，把爆炸特效`ShellExplosion`中的`Play On Wake`勾选上，这一属性会让`ShellExplosion`在被创建时就立即播放特效。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/09.png)

为子弹创建一个脚本，进行触发检测：

_Shell.cs_

```csharp
public class Shell : MonoBehaviour {

	public GameObject shellExplosionPrefab;   // 子弹爆炸特效

	// 触发检测，与任何物体碰撞都会爆炸
	void OnTriggerEnter(Collider collider) {
		// 实例化特效
		GameObject.Instantiate(shellExplosionPrefab, transform.position, transform.rotation);

		// 销毁子弹
		GameObject.Destroy(this.gameObject);

		// 调用 Tank 对象下的 TakeDamage 方法，坦克扣血
		if (collider.tag == "Tank") {
			collider.SendMessage("TakeDamage");
		}
	}
}
```

创建一个新的坦克，将它的编号改为`2`，并且修改它的开火键。你可以操控这两个坦克对轰，效果如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/10.png)

### 为坦克添加爆炸效果

这还不够，我们也得为坦克添加爆炸效果。步骤的话与上面的类似，只不过需要在`TankHealth`中调用爆炸特效。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/11.png)

?> `Tank`对象下有一个`TankRenderers`，想要改颜色的话可以替换材质。

### 定时消除爆炸效果

你可能已经注意到了，游戏在运行过程中创建了许多个`ShellExplosion`。为了让这些已经运行过的爆炸特效销毁，我们可以为`ShellExplosion`添加一个新脚本：

_DestroyForTime.cs_

```csharp
public class DestoryForTime : MonoBehaviour {

	public float time;

	void Start() {
		// 爆炸效果结束后销毁
		Destroy(this.gameObject, time);
	}
}
```

`time`的具体数值可以根据`ShellExplosion`的时长决定，这里我填的是`1.5`

## 设置摄像机跟随

---------
游戏的角色一共有两个，当它们的距离越来越远时，我们希望游戏的画面也能够扩大。显然，当前的摄像机设置是无法满足游戏需求的。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/12.png)

为了达成这种效果，可以先将摄像机调整成下面这样：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/13.png)

假设上图是游戏的初始画面，那么此时的摄像机视野与双方坦克间距的比是`0.59`。当双方的距离产生变化时，摄像机的`Size`应该按照这一比例自动进行调整。因此，我们需要为摄像机写一个控制脚本：

_FollowTarget.cs_

```csharp
public class FollowTarget : MonoBehaviour {

	public Transform player1;     // player1 的位置
	public Transform player2;     // player2 的位置

	private Vector3 offset;       // 相机初始位置与两个坦克中心点的偏移量
	private Camera mainCamera;    // 主相机

	void Start() {
		offset = transform.position - (player1.position + player2.position) / 2;
		mainCamera = GetComponent<Camera>();
	}

	void Update() {
		// 当任意对象被销毁后，摄像机将不再跟随
		if (player1 == null || player2 == null) { return; }

		// 相机位置是两个坦克的中心点与偏移量之和
		transform.position = offset + (player1.position + player2.position) / 2;

		// 坦克之间的距离
		float distance = Vector3.Distance(player1.position, player2.position);

		// 0.59f 是摄像机初始视野 / 坦克初始间距
		float size = distance * 0.59f;

		// 调整摄像机的正交视野
		mainCamera.orthographicSize = size;
	}
}
```

代码我都进行了注释，如果对类方法有什么不懂的话可以自行查阅文档。

## 游戏声音

---------
游戏的声音总共有六个：背景音乐、子弹爆炸声、开炮声、坦克爆炸声、坦克引擎声、坦克行驶声。我们一个个来。

首先是背景音乐。新建一个`SoundManager`对象，为其添加`Audio Source`组件，然后添加背景音乐即可。由于背景音乐是不断循环的，所以记得把`Loop`属性勾选上。

其次就是子弹爆炸声。由于子弹会在爆炸后销毁，而如果你使用的是`Audio Source`组件，那么声音也会随着对象的销毁而停止播放。为了解决这一问题，我们可以使用下面的这段代码：

```csharp
// 子弹爆炸声音
public AudioClip shellExplosionAudio;

// 在子弹位置播放爆炸声音
AudioSource.PlayClipAtPoint(shellExplosionAudio, transform.position);
```

`AudioClip`不会随着对象的销毁而销毁，因此也就避免了这一问题。

接下来，就是开炮声和坦克爆炸声。这两个声音与子弹爆炸声类似，可以使用相同的方法处理。

最后就是坦克引擎声和行驶声音了。为`Tank`添加`Audio Source`组件，将坦克引擎声赋值给它，随后修改代码：

_TankMovement.cs_

```csharp
public class TankMovement : MonoBehaviour {

	public float speed = 5;            // 坦克前进速度
	public float angularSpeed = 5;     // 坦克旋转速度，度
	public float number = 1;           // 坦克的编号
	public AudioClip idleAudio;        // 坦克闲置声音
	public AudioClip drivingAudio;     // 坦克行驶声音

	private Rigidbody tankBody;
	private AudioSource audio;

	void Start() {
		tankBody = this.GetComponent<Rigidbody>();
		audio = this.GetComponent<AudioSource>();
	}

	void FixedUpdate() {
		// Player1 的前进由 W、S 控制，Player2 的前进由 up、down 控制
		float v = Input.GetAxis("VerticalPlayer" + number);

		// Player1 的旋转由 A、D 控制，Player2 的旋转由 left、right 控制
		float h = Input.GetAxis("HorizontalPlayer" + number);

		// 前进、后退
		tankBody.velocity = transform.forward * v * speed;

		// 左右旋转
		tankBody.angularVelocity = transform.up * h * angularSpeed;

		// 如果坦克移动了位置，就要播放声音
		if (Mathf.Abs(h) > 0.1 || Mathf.Abs(v) > 0.1) {
			audio.clip = drivingAudio;
			if (audio.isPlaying == false) { audio.Play(); }
		} else {
			audio.clip = idleAudio;
			if (audio.isPlaying == false) { audio.Play(); }
		}
	}
}
```

## 添加UI

---------
坦克的血量是`100`，为了让这一数值能够显示在玩家的面前，我们需要为坦克添加上血条显示。

### 创建圆形进度条

Unity 有现成的进度条 UI，我们需要将默认的条形进度条改为圆形进度条。创建一个`Slider`，该对象下有`Background`、`Fill Area`、`Handle Slide Area`三个部分:

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/14.png)

背景就是一个长条，而填充区域则是用于显示进度。由于我们做的是一个圆形的进度条，所以可以把`Handle Slide Area`删掉。现在，把`Background`和`Fill`的图片替换成圆形。

当然，把图片换成圆形后，UI 的形状依旧是扁长形的。调整`Slider`、`Background`、`Fill`的`stretch`，将其设为上下左右拉伸：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/15.png)

修改`Fill`的`Image Type`属性，将其设为`Filled`。你可以在这里调整进度条的颜色和填充方式：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/16.png)

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/17.png)

### 将血条添加到坦克身上

当前的血条仍然是 2D 形式的，为了让它渲染成 3D，我们需要修改一下它的渲染模式：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/18.png)

接下来把整个画布添加到`Tank`下，重设一下 UI 的位置，这样血条就会附加在坦克上。当然，你会发现血条的尺寸和旋转角度不对，我们需要调整一下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/19.png)

### 修改生命值脚本

_TankHealth.cs_

```csharp
public class TankHealth : MonoBehaviour {

	public int hp = 100;                       // 坦克当前血量
	public GameObject tankExplosionPrefab;     // 坦克爆炸特效
	public AudioClip tankExplosionAudio;       // 坦克爆炸声音
	public Slider hpSlider;                    // 血条

	private int hpTotal;                       // 坦克总血量

	void Start() {
		hpTotal = hp;
	}

	void TakeDamage() {
		if (hp <= 0) { return; }

		// 坦克扣血
		hp -= Random.Range(10, 20);

		// 修改血条进度
		hpSlider.value = (float)hp / hpTotal;

		// 坦克死亡时爆炸
		if (hp <= 0) {
			// 在坦克的位置播放爆炸声音
			AudioSource.PlayClipAtPoint(tankExplosionAudio, transform.position);

			// 实例化爆炸效果
			GameObject.Instantiate(tankExplosionPrefab, transform.position + Vector3.up, transform.rotation);

			// 销毁坦克
			GameObject.Destroy(this.gameObject);
		}
	}
}
```

?> 进度条属性`value`的取值范围是 0~1，我们需要计算出当前血量与总血量的比，然后再修改`value`。

## 最终效果

---------
至此，我们完成了坦克大战的逻辑编写，实现基本的对战功能。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/TankGame/20.png)

?> 建议把血条的位置稍微往上挪一点，避免血条与地面重叠。