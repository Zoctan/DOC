## JSON 格式字段处理

`dcat-admin` 的表单提供了下面几个组件来处理 `JSON` 格式的字段，方便用来处理 `JSON` 格式的对象、一维数组、二维数组等对象。

## 键值对象 (keyValue)

![](https://cdn.learnku.com/uploads/images/202006/21/38389/jwHjbKP9PL.png!large)

如果你的字段存储的是不固定`键`的 `{"field":"value"}` 格式，可以用 `keyValue` 组件:

```php
$form->keyValue('column_name');

// 设置校验规则
$form->keyValue('column_name')->rules('required|min:5');
```

自定义键名以及键值标题翻译

```php
$form->keyValue(...)->setKeyLabel('键名')->setValueLabel('键值');
```

也可以自定义默认结构，以便于新建数据时候自动带入 keyValue 数据的模板

```php
$form->keyValue('price')->default(['cny' => '', 'usd' => ''])->setKeyLabel('币种')->setValueLabel('价格');
```

![](https://cdn.learnku.com/uploads/images/202109/26/13322/AKy1LPElHF.png!large)

## 固定键值对象 (embeds)

![](https://cdn.learnku.com/uploads/images/202006/21/38389/KwSIti7bs5.png!large)

用于处理 `mysql` 的 `JSON` 类型字段数据或者 `mongodb` 的 `object` 类型数据，也可以将多个 `field` 的数据值以 `JSON` 字符串的形式存储在 `mysql` 的字符串类型字段中

适用于有固定键值的 `JSON` 类型字段

```php
$form->embeds('column_name', function ($form) {

    $form->text('key1')->required();
    $form->email('key2')->required();
    $form->datetime('key3');

    $form->dateRange('key4', 'key5', '范围')->rules('required');
})->saving(function ($v) {
    // 转化为json格式存储
    return json_encode($v);
});

// 自定义标题
$form->embeds('column_name', '字段标题', function ($form) {
    ...
});
```

回调函数里面构建表单元素的方法调用和外面是一样的。

## 一维数组 (list)

![](https://cdn.learnku.com/uploads/images/202006/21/38389/aO6OS0sOyv.png!large)

如果你的字段是用来存储 `["foo", "Bar"]` 格式的一维数组，可以使用 `list` 组件:

```php
$form->list('column_name');

// 设置校验规则
$form->list('column_name')->rules('required|min:5');

// 设置最大和最小元素个数
$form->list('column_name')->max(10)->min(5);
```

## 二维数组 (table)

![](https://cdn.learnku.com/uploads/images/202006/21/38389/755BUms334.png!large)

如果某一个字段存储的是 `json` 格式的二维数组，可以使用 `table` 表单组件来实现快速的编辑：

```php
$form->table('column_name', function ($table) {
    $table->text('key');
    $table->text('value');
    $table->text('desc');
})->saving(function ($v) {
    return json_encode($v);
});
```

这个组件类似于 `hasMany` 组件，不过是用来处理单个字段的情况，适用于简单的二维数据。

## 二维数组 (array)

![](https://cdn.learnku.com/uploads/images/202005/15/38389/MejvctX1V7.png!large)

如果某一个字段存储的是 `json` 格式的二维数组，并且字段比较多，可以使用 `array` 表单组件来实现快速的编辑：

```php
$form->array('column_name', function ($table) {
    $table->text('key');
    $table->text('value');
    $table->textarea('desc');
})->saveAsJson();
```