![state & memento](https://pic2.zhimg.com/v2-610d8ba502fc9286077c996ba027003c_1440w.jpg?source=172ae18b)
## C++ · 设计模式 (状态变化=State<状态机>/ Memento<备忘录>)

**"状态变化"模式**

在组件构建过程中，某些对象的状态经常面临变化，如何对这些变化进行有效地管理？同时又维持高层模块的稳定

**典型模式：**

* State
* Memento

## State (状态模式)
### 问题提出
软件构建过程中，某些对象的状态如果改变，其行为也会随之发生变化。比如文档处于只读状态，其支持的行为和读写状态支持的行为就可能完全不同。如何在运行时，根据对象的状态来透明地更改对象的行为？而不会为对象操作和状态转换之间引入紧耦合？

### 状态机模型
![state module code](https://pic4.zhimg.com/80/v2-dc8f0689ca7b878526f793e07de90b2b_1440w.jpg)

状态机模型创建完毕，在每个状态机对象内，每个操作接口在完成自身任务后，都给 pNext指针赋值到下一个State对象指针。接下来需要 Processor类启动定义好的状态机。
![state module code](https://pic4.zhimg.com/80/v2-0043558ddab3941114161fdd523dfbcf_1440w.jpg)

### 结构
![struct](https://pic1.zhimg.com/80/v2-92b9624461a6a76ec63e579438e00734_1440w.jpg)

### 模式定义
允许一个对象在其内部状态改变时改变它的行为。从而使对象看起来似乎修改了其行为。

### 总结
State模式将所有与一个特定状态相关的行为都放在一个State的子类对象中，在对象状态切换时，切换相应的对象；但同时维持State的接口，这样实现了具体操作和状态转换之间的解耦。

为不同的状态引入不同的对象使得状态转换变得更加明确，而且可以保证不会出现状态不一致的情况。因为转换时原子性的---即要么彻底转换过来，要么不转换。
![compare](https://pic3.zhimg.com/v2-b6c1af47d856fca029e52f99db88d95e_r.jpg)

State模式确实和 Strategy模式很像，不过 State模式在状态的轮转方面有更多的建设。
![state vs strategy](https://pic2.zhimg.com/v2-691b1a18859a63fe2d7435c2021fde09_r.jpg)

### 伪代码

```cpp
// 接口函数变为状态对象的行为。
class NetworkState {
public:
	NetworkState* pNext;
	virtual void Operation1() = 0;
	virtual void Operation2() = 0;
	virtual void Operation3() = 0;

	virtual ~NetworkState(){}
}; 

// ① 单个状态对象，用singleton模式
// ② 新增状态，只需要扩展子类

class OpenState: public NetworkState {
	static NetworkState* m_instance;
public:
	static NetworkState* getInstance() {
		if (m_instance == nullptr) {
			m_instance = new OpenState();
		}
		return m_instance;
	}

	void Operation1() {
		// ***
		pNext = CloseState::getInstance();
	}

	void Operation2() {
		// ...
		pNext = ConnectState::getInstance();
	}

	void Operation3() {
		// xxx
		pNext = OpenState::getInstance();
	}
};

class CloseState: public NetworkState {
	static NetworkState* m_instance;
public:
	static NetworkState* getInstance() {
		if (m_instance == nullptr) {
			m_instance = new CloseState();
		}
		return m_instance;
	}

	void Operation1() {
		// ***
		pNext = OpenState::getInstance();
	}

	void Operation2() {
		// ...
		pNext = ConnectState::getInstance();
	}

	void Operation3() {
		// ***
		pNext = CloseState::getInstance();
	}
};

class ConnectState: public NetworkState {
	static NetworkState* m_instance;
public:
	static NetworkState* getInstance() {
		if (m_instance == nullptr) {
			m_instance = new ConnectState();
		}
		return m_instance;
	}

	void Operation1() {
		// ***
		pNext = OpenState::getInstance();
	}

	void Operation2() {
		// ...
		pNext = CloseState::getInstance();
	}

	void Operation3() {
		// xxx
		pNext = ConnectState::getInstance();
	}
};

// 从一个起始状态开始
class NetworkProcessor {
	NetworkState* pState;

public:
	NetworkProcessor(NetworkState* pSt): pState(pSt){}

	void Operation1() {
		// ...
		pState->Operation1();
		pState = pState->pNext;
		// ...
	}

	void Operation2() {
		// ...
		pState->Operation2();
		pState = pState->pNext;
		// ...
	}

	void Operation3() {
		// ...
		pState->Operation3();
		pState = pState->pNext;
		// ...
	}
};
```

## Memento (备忘录模式)
### 问题的提出
在软件构建过程中，某些对象的状态在转换过程中，可能由于某种需要，要求程序能够回溯到对象之前某个时间点的状态。如果使用一些公有接口让其他对象得到此对象的状态，便会暴露对象的细节实现。如何实现对象状态的良好保存和恢复，又不会破坏对象本身的封装性？

### 模拟实现
![simple code](https://pic1.zhimg.com/80/v2-fdd34da51aca2d08b05669c6a0040ae4_1440w.jpg)

### 结构演变
由于Memento在现代编程语言中需要如序列化的支持，并且很多语言本身支持序列化，我们只需要学习它的快照和回滚思想即可。
![struct change](https://pic2.zhimg.com/80/v2-691b1a18859a63fe2d7435c2021fde09_1440w.jpg)


### 模式定义
在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。这样以后就可以将该对象恢复到原先保存的状态了。

### 要点总结
Memento模式的核心是信息隐藏，即Originator 需要向外界隐藏信息，保持其封装性。但同时又需要将状态保持到外界；

但由于现代语言运行时（如C#、Java等）都具有相当的对象序列化支持，因此往往采用效率较高、又容易正确实现的序列化方案来实现 Memento模式。

Memento的拷贝模式当然是深拷贝，然而深拷贝在对象包含指针甚至多级指针的情况下，有多级嵌套的实现问题，需要序列化对象。



## 参考资料
