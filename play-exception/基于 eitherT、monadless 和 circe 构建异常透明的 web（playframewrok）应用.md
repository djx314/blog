# 基于 EitherT、monadless 和 circe 构建异常透明的 web（playframewrok）应用

## 1、异常处理的那些痛

在日常的 Java 开发中，异常的处理是一个永恒不变的主题，几乎在每一处代码都能看见 try-catch-finally 语句。
有时候写了一段代码，由于不知道异常是否会在调用的方法中产生，你通常都会在外面包裹一层 try 语句。
由于方法的层层嵌套，有时候你甚至不知道一个异常被抛出后该干什么，甚至直接不管，任由系统把报错信息返回给用户。
以至于 Spring 等框架会把某些异常转化为 RuntimeException 以屏蔽内部代码复杂性。

在 Scala 中，甚至不做异常检查，异常的处理变成了可选项，可这在一定程度上降低了系统错误提示的友好性，
所以无论是在 Java 的系统或者 Scala 的系统中，你通常都会看见`系统异常，未知错误，请联系管理员`的提示字样。
有时候不过是个用户权限不足或者找不到 ID 对应的数据这些常见问题，但却因为开发的懒惰而变成了一些莫名其妙
的提示。

## 2、解决思路

在经过 @jilen 的一些启发和自己的一些摸索以后，在针对 Web 的 Playframework 的日常开发中，本人总结了一些
异常处理的看法拿出来和大家分享一下。

由于本人主要从事 Web 后端、前端开发，所以本文只针对使用 Playframework 进行开发的情况讨论。

首先要和大家明确两个问题

（1）异常处理需要做的事情。
个人的观点是，在 Web 项目中，编写逻辑代码的人员，异常处理（非逻辑性异常，逻辑相关的异常会在后面讨论）无非就是做两个事情：

a、把异常信息记录在日志上

b、向用户返回错误提示

（2）错误提示信息和错误状态码在什么时候决定。
个人的观点是，错误提示信息和错误状态码在异常发生或逻辑产生分支的那一刻就应该决定，而不是在最外层捕获异常时
才去想到底应该返回什么信息给用户。

如根据用户 ID 返回用户信息这一条逻辑，在 UserTable.filter(...).headOption 为 None 的那一刻就应该决定错误提示
信息为`找不到 ID 为 2333 的用户`和错误代码是 404，而不是专门构造一个 UserNotFoundException 然后在最外层捕获，
更不应该直接提示一个系统错误或者返回 500 页面。

分析一下上面两个条件，简而言之，就是需要在异常发生的时候记录日志，编写异常信息，并且一直保留到最后直到输出给用户，
这个抽象可以用 Either 来表示，而 Scala 需要使用 Future 来表示异步，于是可以用`Futre[Either[ErrorResult, T]]`或者
更准确的`EitherT[Future, ErrorResult, T]`来代表主体数据类型为 T 的数据所被包装的数据类型。而在一般 Web 应用中，
还需要携带一些错误提示信息和返回的状态码，于是作为相应的结果，这个`EitherT[Future, ErrorResult, T]`通常会变为
`EitherT[Future, ErrorResult, TableData[T]]`，但，这，还没是完全版。

单单把`EitherT[Future, ErrorResult, T]`作为一个包装类型在各个方法间传递的话，难免会因为不断的回调和不断的装箱
操作而令得你的代码可读性变得一团糟，给人一种想回到捕捉异常的方式的冲动。在这个时候，本来平时处于辅助状态的一个可以令异步
代码同步化的框架`monadless`（也可以用`each`）在这里就变得必不可少了。

## 3、前期代码准备

为了使这个复杂的类型`EitherT[Future, ErrorResult, T]`使用起来和普通的`Future[T]`的区别降到最小，在这里需要做一些
准备工作。

首先定义一下上面说到的`ErrorResult`和`TableData`

```scala
import play.api.http.Status
case class ErrorResult(msg: String, code: Int = Status.INTERNAL_SERVER_ERROR)

case class TableData[T](code: Int = Status.OK, msg: String = "", count: Option[Int] = Option.empty, data: T)

object TableData {

  import io.circe._
  import io.circe.generic.extras._
  import io.circe.generic.extras.auto._

  private implicit val config: Configuration = Configuration.default.withDefaults

  implicit def tableDataJsonEncoder[T](implicit encoderT: Encoder[T]): Encoder[TableData[T]] = {
    exportEncoder.instance
  }

  def msg(message: String): TableData[Option[String]] = TableData(msg = message, data = Option.empty)
  def msg(message: String, code: Int): TableData[Option[String]] = TableData(code = code, msg = message, data = Option.empty)

}
```

注：`TableData`需要添加一个`Encoder`以加快编译速度和避免在使用`monadless`的时候触发一个与宏相关的 bug。
`ErrorResult`的 code 字段在这里使用了 Http 状态码作为返回状态。

然后我们需要一个工具，把诸如`Option[T]`，`Either[ErrorResult, T]`，`ErrorResult`，`Future[T]`
这些类型转化为`EitherT[Future, ErrorResult, T]`，另外要注意的是，在工具类里面提供了一个`commonCatch`的函数，
在把`Future[T]`转化为`EitherT[Future, ErrorResult, T]`的时候对异常进行了捕获并且处理，只要在恰当的时候使用
`commonCatch`，就可以在调用其他方法的时候不必关心被调用的方法是否会抛出异常。部分工具类代码如下所示（`commonCatch`）
的具体实现请参照 github 样例工程。

```scala
trait CommonValue {

  type Result[T] = Either[ErrorResult, T]
  type ResultF[T] = Future[Either[ErrorResult, T]]
  type ResultT[T] = EitherT[Future, ErrorResult, T]
  val right = EitherT.right[ErrorResult]
  val rightT = EitherT.rightT[Future, ErrorResult]
  def left[A](value: Future[A])(implicit ec: ExecutionContext) = EitherT.left[A](value)
  def leftT[A] = EitherT.leftT[Future, A]
  val fromOption = EitherT.fromOption[Future]
  val fromEither = EitherT.fromEither[Future]
  def commonCatchGen[T](logger: Logger): FutureExceptionCatch = {
    val logger1 = logger
    new FutureExceptionCatch {
      override lazy val logger = logger1
    }
  }
}

object CommonValue extends CommonValue
```

注：由于这里已经提供了`EitherT`的一些特殊化的别名，下文将使用这些别名作为`EitherT`的替代。

然后就需要一个 trait 把刚才提到的这些东西整合起来

```scala
trait ArcherHelper {

  val dbioLess: MonadlessDBIO = MonadlessDBIO.dbioLess
  val etLess: MonadlessEitherTResult = MonadlessEitherTResult.etLess
  val futureLess: MonadlessFuture = MonadlessFuture
  val logger: Logger = LoggerFactory.getLogger(getClass)
  lazy val commonCatch = V.commonCatchGen(logger)
  type ErrorResult = utils.ErrorResult
  val ErrorResult = utils.ErrorResult

  object R {
    def toJson[T](value: V.ResultT[T])(implicit encoder: Encoder[T], ec: ExecutionContext): Future[Json] = {
      value.fold({ errorResult =>
        TableData.msg(errorResult.msg, errorResult.code).asJson
      }, { data =>
        data.asJson
      })
    }

    def toJsonResult[T](value: V.ResultT[TableData[T]])(implicit encoder: Encoder[T], writeable: Writeable[Json], ec: ExecutionContext): Future[Result] = {
      futureLess.lift {
        val tableData = futureLess.unlift(toJson(value))
        val resultF = value.fold({ errorResult =>
          //val tableData = TableData.msg(errorResult.msg, errorResult.code)
          new Results.Status(errorResult.code).apply(tableData)
        }, { data =>
          new Results.Status(data.code).apply(tableData)
        })
        futureLess.unlift(resultF)
      }
    }
  }

  val V: CommonValue = CommonValue

}
```

这里还提供了一个`toJson`和`toJsonResult`的方法，负责把`V.ResultT[T]`转化为 Json 或者 Result，要注意的是，
Result 的状态码是由 TableData 和 ErrorResult 里面的 code 提供的，这也回应了上文
`错误状态码在异常发生或逻辑产生分支的那一刻就应该决定`的说法。

还有这里也提供了`V.ResultT[T]`，`DBIO[T]`,`Future[T]`这 3 种类型的`monadless`工具类，方便调用，具体代码可参照
样例工程。

## 4、应用场景

把一切准备工作做好以后，我们可以假设一个场景，看看经过改装的机制实用不实用。

例如这里有一个查询快递信息的接口，输入快递单号，如果单号存在，则返回快递信息，如果单号不存在，则返回状态码是 404 的
带提示信息的 json 数据。如果发生系统异常，则返回状态码是 500 的带提示信息的 json 数据。

首先是 Service 部分的代码

```scala
def retrieve(orderKey: String): V.ResultT[TableData[Order]] = {
  val action = Order.filter(s => s.orderKey === accountId).result.headOption
  val orderOptEt = commonCatch(db.run(action)) //捕捉异常
  etLess.lift {
    val orderOpt = etLess.unlift(orderOptEt)
    val orderEt = V.fromOption(orderOpt, ErrorResult(msg = "单号不存在", code = Status.NOT_FOUND))
    TableData(data = etLess.unlift(orderEt), count = Option(1))
  }
}
```

可以看到，在有可能出现异常的地方已经被`commonCatch`捕获，并且会返回一个友好的 ErrorResult 和记录下错误日志，
如果单号不存在，`EitherT`也会跳转到`left`而不会继续后面的流程，
当整个流程都是正确运行之后，才会返回最后的`TableData`。

然后可以看看 Controller 的代码

```scala
def retrieve(orderKey: String) = Action.async { implicit request =>
  R.toJsonResult(orderSerivice.retrieve(orderKey))
}
```

可以看到，只需要一个`R.toJsonResult`，即可把`V.ResultT`转化成需要的 json 结果，由于状态码已经保存在`code`字段
里面了（上文观点：响应状态由业务逻辑决定），所以无需在 Controller 决定响应的状态了。

做好之后，前端无论发送什么请求，都可以得到一个标准的 json 响应，只需要在 ajax success 的时候展示数据（`data`字段）
和在 failed 的时候展示错误信息（`msg`字段）就可以了。

然后现在多了一个验证需求，如果订单号只能是 12 位的字符，不是 12 位的字符就直接不做查询操作直接返回验证错误，那该
如何办呢？

我们可以定义一个方法

```scala
def validate(orderKey: String): V.ResultT[Boolean] = {
  if (orderKey.size == 18) {
    V.rightT(true)
  } else {
    V.leftT[Boolean](ErrorResult(msg = "单号格式错误", code = Status.BAD_REQUEST))
  }
}
```

然后 Service 的代码只需要作以下改动即可

```scala
def retrieve(orderKey: String): V.ResultT[TableData[Order]] = {
  val action = Order.filter(s => s.orderKey === accountId).result.headOption
  val orderOptEt = commonCatch(db.run(action)) //捕捉异常
  etLess.lift {
    etLess.unlift(validate(orderKey)): Boolean //改动的地方，进行验证操作
    val orderOpt = etLess.unlift(orderOptEt)
    val orderEt = V.fromOption(orderOpt, ErrorResult(msg = "单号不存在", code = Status.NOT_FOUND))
    TableData(data = etLess.unlift(orderEt), count = Option(1))
  }
}
```

可以看到，这个验证逻辑的加入对原代码的入侵非常的小，没有了原来的 map flatMap 的改变代码结构的缺点，只用一种
类似抛异常方式即可解决问题，而且相对于抛出异常的方式，这里毋须对外层代码（例如 Controller 的代码）
做任何改变即可实现需求，由于验证返回的结果是`V.Result[Boolean]`，所以可以支持数据库查询等异步验证。

但这里要注意的是，返回类型必须为有意义的类型并且在验证的那一步加上明确的类型标识：

```scala
etLess.unlift(validate(orderKey)): Boolean
```

因为这段代码是上下文无关的，所以如果你忘了加`etLess.unlift`则会使这段验证逻辑失去作用，为了做好类型检查，即使这个
Boolean 的值是多余的应该明确的写出来。

## 5、总结

在上述的例子中，我们看到了`EitherT`和`monadless`所带来的对异常、错误处理的一些大幅度的改善，目前这套逻辑已经
使用在一些实际的项目中，对代码的简化发挥了不少的作用，并且这些都是装箱操作，对性能的影响并不太大，反而在很多时候
还能把异常的抛出转化为仅仅一个`V.left`而减少因为抛出异常而读取堆栈信息所造成的性能损耗。

不过，在上文也声明过，这种处理方法只是比较适合在无论什么情况都只需要返回一个状态码和错误信息、实体数据的 Web 应用
中，对于那些需要对错误进行精致处理的系统中局限性较大，算是一种特殊化的异常处理方法吧。
