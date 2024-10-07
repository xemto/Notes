现有一批类，他们有一批相同的方法，但是他们没有共同的父类。
```cpp
struct MoveMsg {
    int x;  
    int y;
    void speak() { 
        std::cout << "Move " <<  x << ", " << y << '\n';
    }
};

struct JumpMsg {
    int height; 

    void speak() {
        std::cout << "Jump " << height << '\n';
    }
};

struct SleepMsg {
    int time; 

    void speak() { 
        std::cout << "Sleep" << time << '\n'; 
    }
};

struct ExitMsg { 
    void speak() {
        std::cout << "Exit" << '\n';
    }
};
```
如果想存储这些类，那么就需要四个集合来操作，太过繁琐。
## 方法1：std::variant
使用`std::variant`作为包装就可以统一管理以上的类型。
```cpp
using Msg = std::variant<MoveMsg, JumpMsg, SleepMsg, ExitMsg>;

std::vector<Msg> msgs;
```
使用`std::variant`可以达到我们的需求，但是 `Msg`类型占据的空间比较大（包装类型中占据空间最大的那个+1）。
## 方法2：新类包装
```cpp
struct MsgBase {
    virtual void speak() = 0;
    virtual ~Msg() = default;
};

std::vector<MsgBase *> msgs;

template <class Msg>
struct MsgImpl : MsgBase {
    Msg msg;

    MsgImpl(Msg msg) : msg(msg) {
    }

    template<class... Args>
    MsgImpl(Args&&... args) : msg(std::forward<Args>(args)...) {
    }

    void speak() override {
        msg.speak();
    }

    std::shared_ptr<MsgBase> clone() {
        return std::make_shared<MsgImpl<MsgBase>>(msg);
    }
}

msgs.emplace_back(new MsgImpl<MoveMsg>(MoveMsg(10, 11)));
msgs.emplace_back(new MsgImpl<MoveMsg>(10, 11));
```
还可以提供一个工厂方法方便创建对象：
```cpp
std::vector<std::shared_ptr<MsgBase>> msgs;

template <class Msg, class... Args>
auto makeMsg(Args&&... args) {
    // warning: 这里不是 std::make_shared<Msg>
    return std::make_shared<MsgImpl<Msg>>(std::forward<Args>(args)...);
}

msgs.emplace_back(makeMsg<MoveMsg>(5, 10));
```
使用C++ 20 判断类是否有该方法
```cpp
void happy() {
    if constexpr (requires {
        msg.happy();
    }) {
        msg.happy();
    } else {
        std::cout << "No happy" << std::endl;
    }
}
```
# 使用工厂方法去除if...else
在原始代码中需要很多`if...else` 代码。
```cpp
struct RobotClass {
    void recv_data() {
        int type = 0;
        std::cin >> type;

        if (type == 1) {
            is_move = true;
            mov = std::make_shared<MoveMsg>(1, 2);
        }
        if (type == 2) {
            is_jmp = true;
            jmp = std::make_shared<JumpMsg>(1);
        }
    }

    void update() {
        if(is_mov) {
            mov->speak();
        }
        // ...
    }
    std::shared_ptr<MoveMsg> mov;
    std::shared_ptr<JumpMsg> jmp;
    std::shared_ptr<SleepMsg> slp;
    std::shared_ptr<ExitMsg> ext;

    bool is_move;
    bool is_jmp;
    bool is_slp;
    bool is_ext;
}
```
现在我们重构一下：
```cpp
struct MsgBase {
    virtual void speak() = 0;
    virtual ~Msg() = default;

    using Ptr = std::shared_ptr<MsgBase>;	// Add
};

struct RobotClass {
    void recv_data() {
        int type = 0;
        std::cin >> type;

        if (type == 1) {
            msg = makeMsg<MoveMsg>(1, 2);
        }
        if (type == 2) {
            msg = makeMsg<JumpMsg>(1);
        }
        //...
    }
    void update() {
        msg->speak();
    }
    MsgBase::Ptr msg;
};
```
现在还是不够优雅，存在一大堆的if...else，继续优化：
```cpp
template <class Msg>
auto makeMsg() {
    // warning: 这里不是 std::make_shared<Msg>
    return std::make_shared<MsgImpl<Msg>>();
}


struct MsgFactoryBase {
    virtual MsgBase::Ptr create() = 0;
    virtual ~MsgFactoryBase() = default;

    using Ptr = std::shared_ptr<MsgFactoryBase>;
}

template <class Msg> 
struct MsgFactoryImpl : MsgFactoryBase {
    MsgBase::Ptr create() override {
        return std::make_shared<MsgImpl<Msg>>();
    }
}

template <class Msg>
MsgFactoryBase::Ptr makeFactory() {
    return std::make_shared<MsgFactoryImpl<Msg>>();
}

struct RobotClass {
    inline static const std::map<int, MsgFactoryBase::Ptr> lut = {
        {1, makeFactory<MoveMsg>()},
        {2, makeFactory<JumpMsg>()},
        {3, makeFactory<SleepMsg>()},
        {4, makeFactory<ExitMsg>()},
    };
    void recv_data() {
        int type = 0;
        std::cin >> type;
        try {
            msg = lut.at(type)->create();
        } catch (std::out_of_range &) {
            std::cout << "no such msg type!\n";
            return;
        }
    }
}
```
再继续使用宏定义简化：
```cpp
inline static const std::map<std::string, MsgFactoryBase::Ptr> lut = {
#define PER_MSG(Type) {#Type, makeFactory<Type##Msg>()},
        PER_MSG(Move)
        PER_MSG(Jump)
		PER_MSG(Sleep)
        PER_MSG(Exit)
#undef PER_MSG
    // {"Move", makeFactory<MoveMsg>()},
    // {"Jump", makeFactory<JumpMsg>()},
    // {"Sleep", makeFactory<SleepMsg>()},
    // {"Exit", makeFactory<ExitMsg>()},
};
```