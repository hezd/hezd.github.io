---
title: Kotlin Flow
date: 2023-08-25 17:58:21
tags:
- flow
categories:
- Kotlin
---

# Flow

Flow是基于流的编程模型。更简单的说Flow代表数据流。

数据流有三个重要角色

producer：数据生产者

imediatly：中介，可以修改数据流

comsumer：数据消费者

案例分析

pancho（潘乔）需要到湖边打水，以前他都需要每次提桶到湖边打完水返回驻地。可能有时候到湖边发现湖水已经干涸。后来他发现他可以架设管道从湖边到驻地，这样他只需要打开水龙头就可以收集到水流了，并且可以观察到湖水是否已经干涸。

这里的水流可以对应到应用中的数据流



## 响应式编程

像上面案例分析中这种观察者可以对被观察对象变化做出响应的系统称为响应式编程

下面以一个定时器案例进行数据流创建、转换、收集

## 创建数据流

使用flow构建器方法创建

```kotlin
class DataFlowViewModel : ViewModel() {
    val countFlow = flow{
        var count = 0
        while (true) {
            println("emit data:$count")
            emit(count)
            delay(1000)
            count++
        }
    }
}
```

## 修改数据流

使用中间操作符例如map进行转化，

```kotlin
class DataFlowViewModel : ViewModel() {
    val countFlow = flow{
        var count = 0
        while (true) {
            println("emit data:$count")
            emit(count)
            delay(1000)
            count++
        }
    }.map {
      it.toString()
    }
}
```

## 从数据流中收集

通过终端操作符collect进行数据收集

```kotlin
class DataFlowActivity : AppCompatActivity() {
    private val binding by lazy {
        ActivityDataFlowBinding.inflate(layoutInflater)
    }

    private val viewModel : DataFlowViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val root = binding.root
        setContentView(root)

        startCollect()
    }

    private fun startCollect() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED){
                viewModel.countFlow.collect {
                    println("collect data:$it")
                    binding.textview.text = it.toString()
                }
            }
        }
    }
}
```

collect会触发数据发送以及数据流修改，并最终收集到数据

> 注：协程中将这种按需创建并在被观察时才会发送数据的的数据流叫做冷流

## 在Android视图中收集数据流

通常我们是在Android视图中收集数据流，这时我们要关注两个方面，后台运行时不应该浪费资源和配置变更

### 安全收集

安全收集的目的是页面置于后台的时候不应该浪费资源，这里所说的浪费资源是指countFlow属于冷流当置于后台时如果收集任务不取消后台生产者还可以持续的发送数据浪费内存资源。Google官方的建议是使用repeatOnlifecycle例如

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED){
        viewModel.countFlow.collect {
            println("collect data:$it")
            binding.textview.text = it.toString()
        }
    }
}
```

当页面进入后台时收集任务会被取消

另外还提到有launchWhenStarted，页面进入后台后会挂起收集任务而不是取消，可能会造成一定内存开销

```kotlin
lifecycleScope.launchWhenStarted {
    repeatOnLifecycle(Lifecycle.State.STARTED){
        viewModel.countFlow.collect {
            println("collect data:$it")
            binding.textview.text = it.toString()
        }
    }
}
```



### 配置变更

当配置变更时例如屏幕旋转会触发Activity重启，而ViewModel却会保留，使用冷流会导致重新收集数据创建新的数据流，造成资源浪费。有没有一种机制可以保证不管Activity重启多少次都不需要重新创建数据流呢。可以使用热流StateFlow，热流发送数据的活动独立于观察者，当重新触发收集数据时并不会创建新的数据流，并保留了重建前的数据。

但是新的问题来了，对于上面短时间内重启情况下热流对我们很有用，但是如果按下Home键应用回到后台后，虽然收集任务被取消，但是由于热流的特性，后台可能还在继续发送数据，造成资源浪费。

对于这个问题可以使用超时机制，利用stateIn运算符将冷流转为热流同时设置收集取消后数据发送等待多久自动取消

```kotlin
val countFlow = flow{
    var count = 0
    while (true) {
        println("emit data:$count")
        emit(count)
        delay(1000)
        count++
    }
}.map {
  it.toString()
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000),0)
```

如果短时间内重启不会重建数据流，而长时间置于后台触发超时机制会自动取消数据发送,大功告成perfect😁



## stateFlow

stateFlow是热数据流，负责更新stateFlow的类是生产者，收集数据的类是消费者，数据收集不会触发生产者数据发送。

可使用stateFlow替换项目中的LiveData，但请注意StateFlow热流特性，页面销毁并不会停止数据发送

基本用法：

```kotlin
class LatestNewsViewModel(
    private val newsRepository: NewsRepository
) : ViewModel() {

    // Backing property to avoid state updates from other classes
    private val _uiState = MutableStateFlow(LatestNewsUiState.Success(emptyList()))
    // The UI collects from this StateFlow to get its state updates
    val uiState: StateFlow<LatestNewsUiState> = _uiState

    init {
        viewModelScope.launch {
            newsRepository.favoriteLatestNews
                // Update View with the latest favorite news
                // Writes to the value property of MutableStateFlow,
                // adding a new element to the flow and updating all
                // of its collectors
                .collect { favoriteNews ->
                    _uiState.value = LatestNewsUiState.Success(favoriteNews)
                }
        }
    }
}

// Represents different states for the LatestNews screen
sealed class LatestNewsUiState {
    data class Success(val news: List<ArticleHeadline>): LatestNewsUiState()
    data class Error(val exception: Throwable): LatestNewsUiState()
}
```

