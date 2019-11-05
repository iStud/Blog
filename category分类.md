# Category

## 作用

category 也就是分类，作用是在不改原来类的基础上为类增加一些方法。

## 原理

分类的底层是一个结构体，分类中的数据都保存在结构体中，类的每一个分类都对应着一个结构体对象。

```c++
struct _category_t {
	const char *name;
	struct _class_t *cls;
	const struct _method_list_t *instance_methods;
	const struct _method_list_t *class_methods;
	const struct _protocol_list_t *protocols;
	const struct _prop_list_t *properties;
};
```

查看[oc源码](https://opensource.apple.com/tarballs/objc4/)

截取其中片段，该方法的作用是将方法、属性、协议从分类附加到类中。

```c++
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}

```

一般的类它的对象方法、属性、协议、成员变量保存在类对象中，类方法保存在元类对象（meta-class）中。

分类也是如此，通过runtime 动态将分类中的方法合并到类对象和元类对象中。

## 同扩展的区别

类扩展在编译的时候，它的数据已经包含在类的信息中

分类是在运行时才会将数据合并到类信息中

# load 方法和 initialize 方法

## load方法

load 方法是根据方法地址直接调用

load 方法是runtime加载类、分类的时候调用

每个类、分类的 load 方法会在程序运行过程中只调用一次

先调用父类的load方法，然后调用子类的load方法，最后调用分类的load方法。先编译先调用的原则。

## initialize方法

initialize 方法是根据 objc_msgSend 调用

initialize方法会在类第一次接受到消息时调用

先调用父类的initialize方法，在调用子类的initialize方法

如果子类没有实现initialize方法会调用父类的initialize方法，如果分类实现了initialize方法就会覆盖类本身的initialize方法







