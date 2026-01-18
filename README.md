# 刚体物理：Verlet单摆到绳索约束探索，公式推导+代码一条龙

 
## 🌟 01. 导言

Hello大家好啊，又是许久不见诸位，不知道大家有没有想我。今天要讲解的是刚体物理中的一些经典案例，诸如单摆、多摆、绳索、铰链等，我会循序渐进的将这些案例串联起来。总的来说呢，相当于了解并实践一下Unity中的Hinge Joint系统。在此之前先聊一些闲话和提示，如果不想听我废话可以直接去到视频第二章。

我的粉丝们应该都是专业相关度很高的群体，所以我会尽可能代入我初学时的状态，让大家用最小的代价尽可能搞懂，并且如果太过基础的技术，大家可能也学不到真东西，也正因如此，视频的更新速度会很慢。但是我做视频会尽可能严谨、追求高质量。如果催更的诸位有什么想要了解的知识，可以告诉我一些材料或者参考，有价值我会做成视频给大家分享。还有就是我在考虑做一些游戏相关经验访谈视频，可能会去找一些身边的大佬们去谈技术，这个计划还在筹备中，我会单独出一期视频探讨这件事。感兴趣的可以打分、弹幕、评论让我看到。

今天的视频课程涉及刚体模拟、高中物理、微积分、数值积分进阶、unity Hinge Joint系统等前置需求，深度不会超过之前视频提到的XPBD模拟这一课程。请大家备好小板凳，我们将从最简单的单摆（Simple Pendulum）开始，一步步升级到复杂的多摆（Double/Multi Pendulum），最终掌握绳索（Rope）模拟的底层算法。
 
## 📚 02. 单摆（Simple Pendulum）

### 2.1. 现实生活中的“摆”

想象一下：你家墙上的挂钟、游乐园里的海盗船、甚至你把耳机线甩来甩去时的样子——它们都在做着某种形式的“摆动”。这是我们理解复杂物理世界的第一步。
在理想状态下（无阻力、质点、轻绳），单摆的运动遵循牛顿第二定律。再简化一下，就是一个刚体在围绕一个质点做相对运动。
### 2.2. 单摆的数学模型

我们关注的是切线方向上的力 $F_t$：
$$F_t = m a_t$$
其中，$a_t$ 是切向加速度。从几何上看，切向力是重力 $mg$ 的分量。
- **重力的切向分量**：$F_t = -mg \sin\theta$ (负号表示回复力，方向与 $\theta$ 增加方向相反)。
- **$\theta$对时间求导是角速度**，二次导数是加速度，$\theta'' =  \frac{d^2\theta}{dt^2}$。
- **角加速度 $\alpha$ 与切向加速度 $a_t$ 的关系**：$a_t = L \alpha = L \frac{d^2\theta}{dt^2}$ ($L$ 是绳长)。

因此，我们得到单摆的**微分方程**：
$$m L \frac{d^2\theta}{dt^2} = -mg \sin\theta$$
简化后得到：
$$\frac{d^2\theta}{dt^2} = - \frac{g}{L} \sin\theta$$

当角度 $\theta$ 很小时（比如小于 $1^\circ$），我们通常使用**小角度近似**：$\sin\theta \approx \theta$。
微分方程简化为：
$$\frac{d^2\theta}{dt^2} \approx - \frac{g}{L} \theta$$
这是一个**简谐振动（Simple Harmonic Motion, SHM）**方程，解是 $\theta(t) = A \cos(\omega t + \phi)$，其中角频率 $\omega = \sqrt{g/L}$。


### 2.3. 数值积分：告别微积分 (Euler 法)

在计算机中，我们不能直接解微分方程，需要用**数值积分**（Numerical Integration）。最简单的是**欧拉法（Euler Method）**。
假设我们有角度 $\theta$ 和角速度 $\omega = \frac{d\theta}{dt}$：
1. **计算角加速度 $\alpha$**：
	$$\alpha = \frac{d^2\theta}{dt^2} = - \frac{g}{L} \sin\theta$$
2. **更新角速度**：
	$$\omega_{new} = \omega_{old} + \alpha \cdot \Delta t$$
3. **更新角度**：
	$$\theta_{new} = \theta_{old} + \omega_{new} \cdot \Delta t \quad$$
4. **更新位置**：
	$$\text{position} = \text{lastPosition} + (L \sin\theta_{new}, -L \cos\theta_{new}, 0)$$

(注意：这里还有一个小技巧，这里使用了改进的半隐式欧拉，更稳定。我之前在讲XPBD模拟时也提到过这个问题，感兴趣的可以去看那一期视频。关于半隐式欧拉还有一个系统性的讲解视频：https://www.youtube.com/watch?v=nCg3aXn5F3M&t=573s)

### 2.4. Unity 代码实现（单摆）

C#
```c#
using UnityEngine;

/// <summary>
/// Ideal simple pendulum simulation (no Rigidbody, no Joint)
/// Symplectic Euler integration
/// </summary>
public class SimplePendulum : MonoBehaviour
{
	[Header("Pendulum Parameters")]
	public float L = 10f;          // 绳长
	public float g = 9.8f;         // 重力加速度

	[Tooltip("Initial angle (degrees)")]
	public float initialAngle = 30f;

	[Range(0f, 1f)]
	public float damping = 0.001f; // 阻尼

	[Header("Reference")]
	public Transform bob;          // 摆锤物体

	// state
	private float theta;           // 角度（弧度）
	private float omega;           // 角速度

	private Vector3 pivotPosition;

	void Start()
	{
		// Pivot 就是当前物体
		pivotPosition = transform.position;

		// 初始化角度（度 → 弧度）
		theta = initialAngle * Mathf.Deg2Rad;
		omega = 0f;

		UpdateBobPosition();
	}

	void FixedUpdate()
	{
		float dt = Time.fixedDeltaTime;

		// 1. 角加速度
		float alpha = -(g / L) * Mathf.Sin(theta);

		// 2. 更新角速度（Symplectic Euler）
		omega += alpha * dt;
		omega *= (1f - damping);

		// 3. 更新角度
		theta += omega * dt;

		// 4. 更新摆锤位置
		UpdateBobPosition();
	}

	void UpdateBobPosition()
	{
		Vector3 deltaPosition = new Vector3(
			L * Mathf.Sin(theta),
			-L * Mathf.Cos(theta),
			0f
		);

		bob.position = pivotPosition + deltaPosition;
	}

#if UNITY_EDITOR
	void OnDrawGizmos()
	{
		if (bob == null) return;
		Gizmos.color = Color.white;
		Gizmos.DrawLine(transform.position, bob.position);
	}
#endif
}

```

----

##  03. 进阶：多摆与绳索的底层逻辑

### 3.1. 从单摆到双摆
单摆用数值求解的方式尚且还算简单，一起尝试一下双摆？我们运用一些小学二年级的知识很容易得到：
设：
- 质量：m1, m2

- 长度：L1, L2

- 角度：θ1, θ2（相对竖直向下）

- 角速度：ω1, ω2


**标准无阻尼双摆方程：**
$$\begin{aligned}{\theta}_1'' &=\frac{ -g (2m_1 + m_2)\sin\theta_1 - m_2 g \sin(\theta_1 - 2\theta_2)
 - 2 m_2 \sin(\theta_1 - \theta_2) \left(  \omega_2^2 L_2  + \omega_1^2 L_1 \cos(\theta_1 - \theta_2) \right)}{ L_1 \left(  2m_1 + m_2  - m_2 \cos(2\theta_1 - 2\theta_2) \right)} \\[10pt]
{\theta}_2'' &=
\frac{ 2 \sin(\theta_1 - \theta_2) \left(  \omega_1^2 L_1 (m_1 + m_2)  + g (m_1 + m_2) \cos\theta_1
   + \omega_2^2 L_2 m_2 \cos(\theta_1 - \theta_2)\right)}{ L_2 \left( 2m_1 + m_2 - m_2 \cos(2\theta_1 - 2\theta_2) \right)
}
\end{aligned}$$
别笑，你试你也过不了第二关。要不我们换种思路？

**多摆（Double Pendulum）的微分方程极其复杂，具有混沌特性**。我们再想想这个运动有什么特殊性吗？如果将摆锤看作是一个质点，那么双摆就可以看作是一个链式结构，每个摆锤都被**约束（Constraint）**在它上一级摆锤的圆周上。
- **单摆**：一个质点 $P_1$ 被约束在距固定点 $P_0$ 距离 $L_1$ 的圆周上。
- **双摆**：第二个质点 $P_2$ 被约束在距 $P_1$ 距离 $L_2$ 的圆周上。

直接解复杂的微分方程组我不擅长，只考虑当前的摆和它上一级的摆的位置情况，我们还不是手拿把掐？
### 3.2. 基于位置的约束求解：Verlet Integration (公式探索与优化)

传统的欧拉法（或更高级的Runge-Kutta）是基于**速度**和**加速度**的。但模拟链条或绳索时，我们只需要处理“距离不变”这个硬性条件。上古大能早已想到，并且提出了一种算法：**Verlet Integration (维尔莱积分)** ，这里我需要强调这种思想，数学上很多解的思路都是对的，但对的不意味着就要一股脑计算到底，不然只会陷入思维泥潭中。
就像是人生一样，当你幻想一生，在无数纷扰中为了探求心中的目标而迷茫，不如从今天出发换个角度，在平凡中悄然改变。 ————不是鲁迅说的
1. 基本位置更新：

	位置信息是最显而易见的信息，有位置，那么速度和受力也很容易推导。因此Verlet 不直接存储速度 $v$，而是存储上一帧的位置 $x_{old}$当前位置 $x$。Verlet积分基于泰勒展开，推导如下：
	
	在连续时间下，粒子的运动满足牛顿第二定律：
	$$x''(t) = a(t)$$
	对时间进行二阶有限差分近似：
	$$x''(t) \approx\frac{x_{t+\Delta t} - 2x_t + x_{t-\Delta t}}{\Delta t^2}$$
	整理可得 Verlet 更新公式：
	$${x_{t+\Delta t}= 2x_t - x_{t-\Delta t} + a \Delta t^2}$$
	在实现时，通常改写为：
	$$x_{new} = x + (x - x_{old}) + a \Delta t^2$$
	也可以写成：
	$$\boxed{P_i^{*}=P_i+(P_i - P_i^{old})+\vec g \Delta t^2}$$
	注意：这个公式中不显含速度，速度可以通过下式计算：
$$
	\mathbf{v}(t)
	= \frac{\mathbf{x}(t + \Delta t) - \mathbf{x}(t - \Delta t)}{2\Delta t}
	$$
我们很容易写出来代码：
```c#
Vector2 deltaP = pos - prevPos; //这里的deltaP也可以看做是velocity
pos = pos + deltaP + gravity * dt * dt;
```
2. 施加约束 (Constraint Solving)：

	此时得到的 $P_i^{*}$ 只是一个“猜测位置”，它很可能已经破坏了绳长约束。到这里为止，Verlet积分让我们知道了如何预测质点下一次的运动，而此时我们需要请出约束来规定质点的运动方式。整个物理系统，本质上是在预测 → 修正 → 预测 → 修正 的循环中不断逼近真实运动。

	对任意相邻质点 $P_i$ 与 $P_j$，其距离约束可以表示为：
$$C(P_i, P_j) = \lVert P_j - P_i \rVert - L = 0$$

	由于 Verlet 预测只考虑了外力作用，预测位置通常会破坏该约束，因此我们定义当前时刻的约束误差为：
$$C_{ij} = \lVert P_j - P_i \rVert - L$$

	为了修正这一误差，我们只允许粒子沿两点连线方向进行位移。设修正方向的单位向量为：
$$
	\hat{n} = \frac{P_j - P_i}{\lVert P_j - P_i \rVert}
	$$

	在等质量假设下，约束误差在两个粒子之间平均分配，则位置修正公式可以写成：
$$
	P_i \leftarrow P_i + \frac{1}{2} C_{ij} \hat{n}
	$$
$$
	P_j \leftarrow P_j - \frac{1}{2} C_{ij} \hat{n}
	$$

	若其中一个粒子为固定支点（例如绳索的起点或摆的悬挂点），则修正量全部施加在非固定粒子上：
$$
	P_j \leftarrow P_j - C_{ij} \hat{n}
	$$


	在双摆模拟中，我们有两个质点，每个质点受到重力和杆的约束力。我们可以先使用Verlet积分更新位置，而不考虑约束力，然后通过校正位置使得两点之间的距离等于杆长。这种方法称为“位置校正”或“投影”。

	具体步骤：

	1、对于每个质点，计算所受的外力（如重力），从而得到加速度。

	2、使用Verlet积分公式计算每个质点的预测位置（临时位置）。

	3、根据约束条件（杆长固定）校正预测位置，使得两个质点之间的距离等于杆长。

	4、更新位置。

	对于双摆，我们有两个质点，所以需要分别处理每个质点，但校正时需要考虑两个质点之间的约束。

	更一般地，对于约束系统，Verlet积分常与“约束投影”结合，形成所谓的“Verlet积分约束求解”方法，在分子动力学和物理引擎中广泛应用。

下面我们详细说明如何校正。

假设两粒子 $P_i$ 和 $P_{i+1}$ 之间的目标距离是 $D$。当前距离是 $d = |P_i - P_{i+1}|$。
**目标**：将 $P_i$ 和 $P_{i+1}$ 沿连线方向拉近或推远，使得它们之间的距离回到 $D$。
- **连线方向向量** $\vec{V} = P_{i+1} - P_i$
- **误差** $e = d - D$
- **校正量** $S = \frac{e}{2d} \cdot \vec{V}$
**更新位置**：
$$P_i^{new} = P_i + S$$
$$P_{i+1}^{new} = P_{i+1} - S$$
**优化**：为了保证链条不散架，我们需要在一个时间步内**迭代（Iterate）多次执行这个约束求解，直到所有连接都接近目标距离。这被称为迭代松弛（Iterative Relaxation）这也是Position-Based Dynamics (PBD)** 的核心思想。


C#
```c#
// 双摆示例
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// Double Pendulum using Verlet Integration + Distance Constraints (PBD)
/// </summary>
public class VerletDoublePendulum : MonoBehaviour
{
	[Header("Physical Parameters")]
	public float L1 = 2f;
	public float L2 = 2f;
	public Vector2 gravity = new Vector2(0, -9.81f);

	[Header("Solver")]
	[Range(1, 20)]
	public int constraintIterations = 6;

	[Header("Visualization")]
	public Transform bob1;
	public Transform bob2;

	struct Particle
	{
		public Vector2 pos;
		public Vector2 prevPos;
	}

	struct Constraint
	{
		public int i, j;
		public float restLength;
	}

	List<Particle> particles = new List<Particle>();
	List<Constraint> constraints = new List<Constraint>();

	Vector2 pivot;

	void Start()
	{
		pivot = transform.position;

		// --- Initialize particles ---
		Particle p0 = new Particle
		{
			pos = pivot,
			prevPos = pivot
		};

		Particle p1 = new Particle
		{
			pos = pivot + new Vector2(3f, -L1),
			prevPos = pivot + new Vector2(0.5f, -L1)
		};

		Particle p2 = new Particle
		{
			pos = p1.pos + new Vector2(2f, -L2),
			prevPos = p1.pos + new Vector2(0.5f, -L2)
		};

		particles.Add(p0);
		particles.Add(p1);
		particles.Add(p2);

		// --- Distance constraints ---
		constraints.Add(new Constraint { i = 0, j = 1, restLength = L1 });
		constraints.Add(new Constraint { i = 1, j = 2, restLength = L2 });
	}

	void FixedUpdate()
	{
		float dt = Time.fixedDeltaTime;
		float dt2 = dt * dt;

		// 1. Verlet integration (skip fixed pivot)
		for (int i = 1; i < particles.Count; i++)
		{
			Particle p = particles[i];

			Vector2 velocity = p.pos - p.prevPos;
			Vector2 newPos = p.pos + velocity + gravity * dt2;

			p.prevPos = p.pos;
			p.pos = newPos;

			particles[i] = p;
		}

		// 2. Constraint solver (iterative projection)
		for (int k = 0; k < constraintIterations; k++)
		{
			// Fix pivot
			Particle root = particles[0];
			root.pos = pivot;
			particles[0] = root;

			foreach (var c in constraints)
			{
				Particle pA = particles[c.i];
				Particle pB = particles[c.j];

				Vector2 delta = pB.pos - pA.pos;
				float dist = delta.magnitude;

				if (dist < 1e-6f) continue;

				float error = dist - c.restLength;
				Vector2 correction = delta / dist * error;

				// Equal mass assumption
				if (c.i != 0)
					pA.pos += correction * 0.5f;

				pB.pos -= correction * 0.5f;

				particles[c.i] = pA;
				particles[c.j] = pB;
			}
		}

		// 3. Output
		bob1.position = particles[1].pos;
		bob2.position = particles[2].pos;
	}

#if UNITY_EDITOR
	void OnDrawGizmos()
	{
		if (particles.Count < 3) return;
		Gizmos.color = Color.white;
		Gizmos.DrawLine(particles[0].pos, particles[1].pos);
		Gizmos.DrawLine(particles[1].pos, particles[2].pos);
	}
#endif
}

```

```c#
//N摆示例
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// N-Pendulum using Verlet Integration + Distance Constraints (PBD)
/// </summary>
public class VerletNPendulum : MonoBehaviour
{
	[Header("Pendulum Settings")]
	[Min(1)]
	public int segmentCount = 5;        // N
	public float segmentLength = 1.0f;  // 每节长度
	public Vector2 gravity = new Vector2(0, -9.81f);

	[Header("Solver")]
	[Range(1, 30)]
	public int constraintIterations = 8;

	[Header("Initial Angles (Degrees)")]
	public float initialAngleDeg = 120f;

	[Header("Visualization")]
	public Transform bobPrefab;

	struct Particle
	{
		public Vector2 pos;
		public Vector2 prevPos;
	}

	struct Constraint
	{
		public int i, j;
		public float restLength;
	}

	List<Particle> particles = new List<Particle>();
	List<Constraint> constraints = new List<Constraint>();
	List<Transform> visuals = new List<Transform>();

	Vector2 pivot;

	void Start()
	{
		pivot = transform.position;

		particles.Clear();
		constraints.Clear();
		visuals.Clear();

		float angleRad = initialAngleDeg * Mathf.Deg2Rad;
		Vector2 dir = new Vector2(Mathf.Sin(angleRad), -Mathf.Cos(angleRad));

		// P0：固定支点
		particles.Add(new Particle
		{
			pos = pivot,
			prevPos = pivot
		});

		// P1...PN
		for (int i = 1; i <= segmentCount; i++)
		{
			Vector2 pos = pivot + dir * segmentLength * i;

			particles.Add(new Particle
			{
				pos = pos,
				prevPos = pos
			});

			constraints.Add(new Constraint
			{
				i = i - 1,
				j = i,
				restLength = segmentLength
			});

			// 可视化
			if (bobPrefab != null)
			{
				Transform v = Instantiate(bobPrefab, pos, Quaternion.identity);
				visuals.Add(v);
			}
		}
	}

	void FixedUpdate()
	{
		float dt = Time.fixedDeltaTime;
		float dt2 = dt * dt;

		// 1. Verlet integration (skip pivot)
		for (int i = 1; i < particles.Count; i++)
		{
			Particle p = particles[i];

			Vector2 velocity = p.pos - p.prevPos;
			Vector2 newPos = p.pos + velocity + gravity * dt2;

			p.prevPos = p.pos;
			p.pos = newPos;

			particles[i] = p;
		}

		// 2. Constraint solver
		for (int k = 0; k < constraintIterations; k++)
		{
			// 固定支点
			Particle root = particles[0];
			root.pos = pivot;
			particles[0] = root;

			foreach (var c in constraints)
			{
				Particle pA = particles[c.i];
				Particle pB = particles[c.j];

				Vector2 delta = pB.pos - pA.pos;
				float dist = delta.magnitude;

				if (dist < 1e-6f) continue;

				float error = dist - c.restLength;
				Vector2 correction = delta / dist * error;

				// equal mass
				if (c.i != 0)
					pA.pos += correction * 0.5f;

				pB.pos -= correction * 0.5f;

				particles[c.i] = pA;
				particles[c.j] = pB;
			}
		}

		// 3. Output
		for (int i = 1; i < particles.Count; i++)
		{
			if (i - 1 < visuals.Count)
				visuals[i - 1].position = particles[i].pos;
		}
	}

#if UNITY_EDITOR
	void OnDrawGizmos()
	{
		if (particles.Count < 2) return;

		Gizmos.color = Color.white;
		for (int i = 0; i < particles.Count - 1; i++)
		{
			Gizmos.DrawLine(particles[i].pos, particles[i + 1].pos);
		}
	}
#endif
}

```


----
### �️ 04. 进阶：模拟机械铰链（Hinge Joint）

我们已经实现了绳索，但绳索是完全柔软的。如果我们想要模拟一个**机械臂**、**链条锁**或者**布娃娃（Ragdoll）**的肢体，我们就需要引入**角度约束**。

#### 4.1. 铰链的本质

在物理引擎中，铰链（Hinge）通常指限制两个刚体只能绕特定轴旋转。在我们的粒子系统（PBD）中，这表现为限制**三个连续粒子之间的夹角**。

想象三个粒子 $P_0, P_1, P_2$：
- **距离约束** 保证了它们连成线。
- **角度约束** 保证了这条线不会随意弯曲。

#### 4.2. 极简角度限制算法

为了保持教程的易懂性，我们不使用复杂的雅可比矩阵，而是采用一种直观的**几何修正法**：
1. 获取向量 $\vec{v}_1 = P_1 - P_0$ 和 $\vec{v}_2 = P_2 - P_1$。
2. 计算当前夹角。
3. 如果夹角超出了我们设定的限制（比如 $\pm 45^\circ$），就强制把 $P_2$ 旋转回限制边缘。

#### 4.3. Unity 代码实现（带角度限制的链条）

这是一个带有“角度刚度”的链条模拟，可以用来模拟机械尾巴、起重机臂或者植被摆动。

C#
```c#
using UnityEngine;
using System.Collections.Generic;

/// <summary>
/// Hinge Chain with Angle Limits
/// </summary>
public class VerletHingeChain : MonoBehaviour
{
    [Header("Chain")]
    public int segmentCount = 5;
    public float segmentLength = 1.0f;
    public Vector2 gravity = new Vector2(0, -9.81f);

    [Header("Hinge Settings")]
    [Range(0f, 180f)]
    public float maxAngleLimit = 45f; // 最大弯曲角度
    [Range(0f, 1f)]
    public float stiffness = 0.5f;    // 修正力度 (0=软, 1=硬)

    [Header("Solver")]
    public int iterations = 10;

    [Header("Visuals")]
    public Transform nodePrefab;

    struct Particle { public Vector2 pos, prevPos; }
    List<Particle> particles = new List<Particle>();
    List<Transform> visuals = new List<Transform>();
    Vector2 pivot;

    void Start()
    {
        pivot = transform.position;
        for (int i = 0; i <= segmentCount; i++)
        {
            Vector2 pos = pivot + new Vector2(0, -segmentLength * i);
            particles.Add(new Particle { pos = pos, prevPos = pos });
            if (nodePrefab) visuals.Add(Instantiate(nodePrefab, pos, Quaternion.identity));
        }
    }

    void FixedUpdate()
    {
        float dt = Time.fixedDeltaTime;

        // 1. Verlet Integration
        for (int i = 1; i < particles.Count; i++)
        {
            Particle p = particles[i];
            Vector2 vel = p.pos - p.prevPos;
            p.prevPos = p.pos;
            p.pos = p.pos + vel + gravity * (dt * dt);
            particles[i] = p;
        }

        // 2. Constraints
        for (int k = 0; k < iterations; k++)
        {
            // A. Pin Root
            Particle root = particles[0];
            root.pos = pivot;
            particles[0] = root;

            // B. Distance Constraint
            for (int i = 0; i < particles.Count - 1; i++)
            {
                Particle p1 = particles[i];
                Particle p2 = particles[i+1];
                Vector2 delta = p2.pos - p1.pos;
                float currentLen = delta.magnitude;
                float error = currentLen - segmentLength;
                Vector2 dir = delta.normalized;
                
                if (i != 0) p1.pos += dir * error * 0.5f;
                p2.pos -= dir * error * 0.5f;
                
                particles[i] = p1;
                particles[i+1] = p2;
            }

            // C. Angle Constraint (Hinge Logic)
            SolveAngleConstraints();
        }

        // 3. Update Visuals
        for (int i = 0; i < visuals.Count; i++)
        {
            if (i < particles.Count) 
                visuals[i].position = particles[i].pos;
        }
    }

    void SolveAngleConstraints()
    {
        // 遍历每三个连续粒子 (i-1, i, i+1)
        for (int i = 1; i < particles.Count - 1; i++)
        {
            Particle pPrev = particles[i-1];
            Particle pCurr = particles[i];
            Particle pNext = particles[i+1];

            Vector2 v1 = pCurr.pos - pPrev.pos;
            Vector2 v2 = pNext.pos - pCurr.pos;

            // 计算有符号角度 (2D)
            float angle = Vector2.SignedAngle(v1, v2);

            // 如果角度超出限制
            if (Mathf.Abs(angle) > maxAngleLimit)
            {
                // 目标角度：限制在边界上
                float targetAngle = Mathf.Sign(angle) * maxAngleLimit;
                // 需要旋转的角度差
                float diff = angle - targetAngle;

                // 旋转 v2 向量
                // 这里我们简单地移动 pNext 来满足角度 (近似)
                Quaternion rot = Quaternion.Euler(0, 0, -diff * stiffness);
                Vector2 newV2 = rot * v2;
                
                pNext.pos = pCurr.pos + newV2;
                
                // 写回
                particles[i+1] = pNext;
            }
        }
    }

#if UNITY_EDITOR
    void OnDrawGizmos()
    {
        if (particles.Count == 0) return;
        Gizmos.color = Color.yellow;
        for (int i = 0; i < particles.Count - 1; i++)
            Gizmos.DrawLine(particles[i].pos, particles[i+1].pos);
    }
#endif
}
```

通过调节 `Max Angle Limit`，你可以得到一条像蛇一样扭动的链条，或者像钢筋一样笔直的杆子。


----
### �🚀 05. 展望与更高难度方向

我们的“一条龙”之旅到此告一段落。我们用最基础的力学原理和数值算法，实现了从钢性到柔性的过渡，又通过角度约束找回了刚性。

#### 5.1. 更高的挑战

- **PBD 的优化**：我们只实现了基础的距离和角度约束。一个完整的PBD系统还需要**体积约束**（模拟流体）、**形状匹配**（Shape Matching，模拟软体）等。
- **碰撞检测与响应**：当前的绳索会穿过墙壁。真正的物理引擎需要实现**粒子与几何体的碰撞检测**和**非穿透性响应**。
- **GPU 加速**：当粒子数量达到数万个时（例如模拟大规模布料或毛发），需要将所有计算迁移到**Compute Shader**（计算着色器）中进行并行计算。

#### 5.2. 参考文献与学习资料

如果你想更深入地探索，推荐以下关键词：
1. **Verlet Integration** (粒子运动的基础)
2. **Position-Based Dynamics (PBD)** (约束求解的现代方法)
3. **XPBD (Extended PBD)** (更物理精确的 PBD 变体)
4. **Mass-Spring System** (弹簧-质量系统，另一种常见的柔体模拟方法)


----
### 🎉 06. 结语

好了，今天我们从牛顿的微分方程一路走到了现代的粒子约束算法。希望你现在对 Unity 物理模拟的底层逻辑有了更深刻的理解。
**物理引擎的本质，就是不断地预测位置，然后不断地修正错误。**
如果你觉得这份代码和讲解对你有帮助，别忘了**点赞、订阅**，并把这个视频分享给你身边对物理模拟感兴趣的小伙伴！
