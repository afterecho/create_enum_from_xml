# Generating a Java enum class from an Android attrs XML

## Introduction (and warning)

This is a hack.  Nothing more.  I'm sure someone with more Gradle skills than me can probably turn
it into a proper plugin, but if you need something quick and dirty, try this...but don't blame me
if it screws things up.

## The guts

The guts of the thing are in the module level build.gradle.  Here I'll break it down.

```
android.applicationVariants.all { variant ->
```
This runs against all variants of build that are configured.  For more information about variants,
see https://developer.android.com/tools/building/configuring-gradle.html

```
File outdir = new File("${project.buildDir}/generated/source/attrenum/${variant.dirName}")
File attrsfile = new File("${project.projectDir}/src/main/res/values/attrs.xml")
```
Two File objects are defined.  `outdir` is the base directory where the generated Java file will be
placed.  I have hardcoded some of the path (`generated/source`) as I don't know if it is available
in some other way.  The `attrenum` is chosen by me.  You can choose anything you like but don't
use something that is already in use.  (Have a look in the generated/source directory after you've
done a build to see what is already in use.)

`attrsfile` is where the XML with the declared styleable enums are.  As you can see I've hardcoded
a large chunk of the path as I don't know any other way to get it.

```
def attrEnumTask = tasks.create("attrenum${variant.name}") << {
```
This creates a new task.  The `${variant.name}` makes the task name unique per build variant
otherwise you'll get complaints.

```
def attrs = new XmlParser().parse(attrsfile)
def styleable = attrs.depthFirst().find {
    it.name() == "declare-styleable" && it.@name == "MyEnumStyle"
}
def enumlist = styleable.depthFirst().find { it.name() == "attr" && it.@name == "fubar" }
def vals = enumlist.depthFirst().findAll { it.name() == "enum" }
```
Ok, not very pretty, but it parses the attrs.xml file, finds the `declare-styleable` block that
has a name of MyEnumStyle and then finds the attr element that has a name of fubar and then gets
all the enums within.  I have no doubt that there are cleaner, prettier ways, but I can't do
all the work for you.  :)

```
com.squareup.javapoet.TypeSpec.Builder attrEnumBuilder = com.squareup.javapoet.TypeSpec.enumBuilder("AttrEnum")
        .addModifiers(javax.lang.model.element.Modifier.PUBLIC)
vals.each {
    attrEnumBuilder.addEnumConstant(it.@name, com.squareup.javapoet.TypeSpec.anonymousClassBuilder('$L', it.@value as int).build())
}

attrEnumBuilder
        .addField(com.squareup.javapoet.TypeName.INT, "index", javax.lang.model.element.Modifier.PUBLIC, javax.lang.model.element.Modifier.FINAL)
        .addMethod(com.squareup.javapoet.MethodSpec.constructorBuilder()
        .addParameter(com.squareup.javapoet.TypeName.INT, "index")
        .addStatement('this.$N = $N', "index", "index")
        .build())

com.squareup.javapoet.TypeSpec attrEnum = attrEnumBuilder.build();

com.squareup.javapoet.JavaFile javaFile = com.squareup.javapoet.JavaFile.builder("com.afterecho.android.util", attrEnum).build();
```
This uses Square's [JavaPoet](https://github.com/square/javapoet) to create an enum.  Each enum is named after the attribute `name` in the
XML so make sure you use valid Java identifiers.  I've also given the enum a parameter which is the
`value` attribute in case you don't use consecutive values.  JavaPoet is very powerful so you can
do so much more than this if you wish.  Or replace it all with a load of printlns if you want.

The string parameter in the builder() call is the package name for your class.

```
javaFile.writeTo(outdir)
```
This writes the Java source file.  Luckily it also creates the package directory structure too.

```
attrEnumTask.description = 'Turns a stylable enum into a Java enum'
variant.registerJavaGeneratingTask attrEnumTask, outdir
```
These last two lines give the task a description and register it.

Once you have all this, a rebuild of your project and Hey Presto!  There's a new Java file that
looks something like this:

```
package com.afterecho.android.util;

public enum AttrEnum {
  FirstEnum(0),

  SecondEnum(1),

  ThirdEnum(2),

  FourthEnum(3);

  public final int index;

  AttrEnum(int index) {
    this.index = index;
  }
}
```
You can see it in AndroidStudio through the normal method of opening Java files (CTRL-N on Linux).
You'll notice when you open it that there's the warning "Files under the build folder are generated
and should not be edited."  That shows it's in the right place.

Use as you see fit!

No licence.  Public Domain.  No rights reserved.  E&OE.  Contents may settle in transit.
Not responsible for any damage the code does.  For novelty purposes only.  Not to be taken
internally.

