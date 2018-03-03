# GdxPlugin


GdxPlugin is a Gradle plugin that adds two [LibGDX](https://libgdx.badlogicgames.com/) related tasks for use in build files:

* `PackTextures` for creating texture packs (a.k.a. texture atlases) using LibGDX's [TexturePacker](https://github.com/libgdx/libgdx/wiki/Texture-packer) 
* `DistanceField` for creating distance fields from single images using LibGDX's 
[DistanceFieldGenerator](https://github.com/libgdx/libgdx/wiki/Distance-field-fonts#using-distance-fields-for-arbitrary-images)

**This plugin requires Gradle 3.0 or higher**

# Table of Contents

- [Getting started](#getting-started)
- [PackTextures task](#packtextures-task)
  - [Settings](#settings)
  - [Generating multiple texture packs](#generating-multiple-texture-packs)
  - [Reusing settings](#reusing-settings)
  - [Multiple input directories, filtering and renaming](#multiple-input-directories-filtering-and-renaming)
  - [Using "pack.json"](#using-packjson)
  - [Custom tasks](#custom-tasks)
- [DistanceField task](#distancefield-task)
  - [Arguments](#arguments)
  - [DistanceField and PackTextures](#distancefield-and-packtextures)
- [General](#general)
  - [LibGDX version](#libgdx-version)
- [Changelog](#changelog)
  - [1.0.1](#101)

# Getting started
Add the plugin to your project:

```kotlin
plugins {
    id("com.github.blueboxware.gdx") version "1.0.1"
}
```

Create a packTextures task:

```kotlin
import com.github.blueboxware.gdxplugin.tasks.PackTextures
import com.github.blueboxware.gdxplugin.dsl.*

val packTextures: PackTextures by tasks

packTextures.apply {

    // The directory which contains the images to pack
    from("textures/")
    
    // The target directory: 'pack.atlas' is placed in this directory
    into("assets/")
    
    settings {
        // Settings for TexturePacker
        filterMin = MipMapLinearNearest
        filterMag = MipMap
    }
    
}
```

Run the task (or just do a build):

```dos
gradlew.bat packTextures
```

To create distance field tasks:

```kotlin
import com.github.blueboxware.gdxplugin.tasks.DistanceField

val distanceFields: NamedDomainObjectContainer<DistanceField> by extensions

distanceFields.invoke {

    // Creates a task called generateLogoDistanceField
    "logo" {
    
        inputFile = file("textures/logo.png")
        downscale = 8
        spread = 32f
        outputFile = file("assets/logo-df.png")
        
    }
    
    // Creates a task called generateTitleDistanceField
    "title" {
    
        inputFile = file("textures/title.jpg")
        downscale = 4
        spread = 16f
        color = "ff0000"
        outputFile = file("assets/title-df.png")
    
    }

}

```

# PackTextures task

## Settings
Settings for Texture Packer are specified in a `settings { }` block. See the [LibGDX Wiki](https://github.com/libgdx/libgdx/wiki/Texture-packer#settings) 
for a list of available settings, their default values and descriptions. To get a quick overview of the available settings you can run the 
`texturePackerSettingsHelp` Gradle task.

For reference, these are the most important settings and their default values, as of LibGDX 1.9.8: 

```kotlin
settings {

    paddingX = 2
    paddingY = 2
    edgePadding = true
    duplicatePadding = false
    rotation = false
    minWidth = 16
    minHeight = 16
    maxWidth = 1024
    maxHeight = 1024
    square = false
    stripWhitespaceX = false
    stripWhitespaceY = false
    alphaThreshold = 0
    filterMin = Nearest
    filterMag = Nearest
    wrapX = ClampToEdge
    wrapY = ClampToEdge
    format = RGBA8888
    alias = true
    outputFormat = "png"
    jpegQuality = 0.9f
    ignoreBlankImages = true
    fast = false
    debug = false
    combineSubdirectories = false
    flattenPaths = false
    premultiplyAlpha = false
    useIndexes = true
    bleed = true
    bleedIterations = 2
    limitMemory = true
    grid = false
    scale = floatArrayOf(1f)
    scaleSuffix = arrayOf("")
    scaleResampling = arrayOf(Resampling.Bicubic)
    atlasExtension = ".atlas"

}
```

## Generating multiple texture packs
If you want to create multiple texture packs, you can use a `texturePacks { }` block.

The following example creates 3 tasks: pack**Game**Textures, pack**Menu**Textures and pack**GameOver**Textures:
```kotlin
import com.github.blueboxware.gdxplugin.tasks.PackTextures
import com.github.blueboxware.gdxplugin.dsl.*

val texturePacks: NamedDomainObjectContainer<PackTextures> by extensions

texturePacks.invoke {

    // Creates "game.atlas"
    "game" {
        from("textures/game")
        into("assets")
    }

    // Creates "menu.atlas"
    "menu" {
        from("textures/menu")
        into("assets")
        
        settings {
            filterMin = MipMapLinearNearest
            filterMag = Nearest
        }
    }

    "gameOver" {
        from("textures/gameOver")
        into("assets")
        // Name the pack "end.atlas" instead of the default "gameOver.atlas"
        packFileName = "end.atlas"  
    }

}

```

## Reusing settings
To reuse settings for multiple texture packs, you can define settings objects with `packSettings { }`:

```kotlin
import com.github.blueboxware.gdxplugin.tasks.PackTextures
import com.github.blueboxware.gdxplugin.dsl.*

// Create base settings
val baseSettings = packSettings {
    filterMin = MipMapLinearNearest
    filterMag = Nearest
    maxWidth = 2048
    maxHeight = 2048
}

// Create settings for scaled texture packs based on the base settings
val scaledPackSettings = packSettings(baseSettings) { 
    scale = floatArrayOf(1f, 2f)
    scaleSuffix = arrayOf("Normal", "Scaled")
    scaleResampling = arrayOf(Resampling.Bicubic, Resampling.Bicubic)
}

val texturePacks: NamedDomainObjectContainer<PackTextures> by extensions

texturePacks.invoke {

  "game" {
        from("textures/game")
        into("assets")
        settings = baseSettings
    }

    "menu" {
        from("textures/menu")
        into("assets")
        settings = scaledPackSettings
    }

    "gameOver" {
        from("textures/gameOver")
        into("assets")
        
        // Use baseSettings, but change outputFormat to jpg
        settings = packSettings(baseSettings) { 
            outputFormat = "jpg"
        }
    }

}

```

## Multiple input directories, filtering and renaming
Pack Textures tasks implement Gradle's [CopySpec](https://docs.gradle.org/current/javadoc/org/gradle/api/file/CopySpec.html), so you can specify
multiple input directories, and filter and rename files:

```kotlin
packTextures.apply {

    into("assets")
    
    from("textures/ui") {
      exclude("test*")
    } 
    
    from("textures/menu") {
      include("*.png")
      rename("menu_(.*)", """$1""")
    }

}

```

## Using "pack.json"
Normally any `pack.json` files in the input directories (and any subdirectories) are ignored. If you want to load the texture packer settings from a 
pack.json file instead of defining them in the build file, you can use the `settingsFile` argument:

```kotlin
packTextures.apply {

  from("textures/")
  into("assets/")
  
  settingsFile = file("textures/pack.json")

}
```

If you want TexturePacker to use pack.json files found in the input directories and any subdirectories, set `usePackJson` to true:

 
```kotlin
packTextures.apply {

  from("textures/")
  into("assets/")
  
  usePackJson = true

}
```

Note that if you specify multiple input directories (see [above](#multiple-input-directories-filtering-and-renaming)), and more than one of the top level directories contain
pack.json files, only one of these is used. Use the `settingsFile` parameter to specify which one.

## Custom tasks
The plugin provides the `PackTextures` task type which can be used to create custom tasks:

```kotlin
import com.github.blueboxware.gdxplugin.tasks.PackTextures
import com.github.blueboxware.gdxplugin.dsl.*

task<PackTextures>("myPackTask") {

  description = "Pack things"

  into("assets")
  from("textures")

  settings {
    atlasExtension = ".pack"
    filterMin = MipMapLinearLinear
  }

  doLast { 
    println("Done!")
  }

}.let {
  tasks.findByName("build")?.dependsOn(it)    
}

```

Note that we added `myPackTask` to the dependencies of the `build` task so that myPackTask is automatically run when building the project. This is not necessary for the plugins builtin tasks (like `packTextures`):
they are automatically added to the build. 

# DistanceField task
## Arguments
The arguments for the distance field task:

* `inputFile`: The input file (type: File)
* `outputFile`: The output file (type: File, default: inputFileWithoutExtension + "-df." + outputFormat) 
* `color`: The color of the output image (type: String, default: "ffffff")
* `downscale`: The downscale factor (type: int, default: 1)
* `spread`: The edge scan distance (type: float, default: 1.0)
* `outputFormat`: The output format (type: String, default: The extension of `outputFile`. If `outputFile` is not specified: "png")

## DistanceField and PackTextures
If the distance fields you create should be packed by one or more pack tasks, you can add the relevant distance field tasks to the dependencies
of the pack tasks, to make sure the distance fields are available and up to date when the pack task runs. You can do this using Gradle's [dependsOn mechanism](https://docs.gradle.org/4.5.1/dsl/org.gradle.api.Task.html#N17778):

```kotlin
val distanceFields: NamedDomainObjectContainer<DistanceField> by extensions

distanceFields.invoke {

    "logo" {
    
        inputFile = file("textures/logo.png")
        outputFile = file("textures/logo-df.png")
        
    }
    
    "title" {
    
        inputFile = file("textures/title.png")
        outputFile = file("textures/title-df.png")
    
    }

}

val packTextures: PackTextures by tasks

packTextures.apply {

  from("textures/")
  into("assets/")
  
  dependsOn("generateLogoDistanceField", "generateTitleDistanceField")
}
```

# General
## LibGDX version
The plugin comes with a bundled version of LibGDX Tools, which is used for packing etc. To see the LibGDX version used by the 
plugin (this is not the version used by your project itself), run the `gdxVersion` task:

```dos
> gradlew.bat -q gdxVersion
1.9.8
```

If you want the plugin to use a different version, you can force this in the `buildscript` block. For example, to use version 1.9.5:

```kotlin
buildscript {
    repositories { 
        mavenCentral()
    }


    configurations.all {
        resolutionStrategy {
            force("com.badlogicgames.gdx:gdx-tools:1.9.5")
        }
    }
}

```

Use the `gdxVersion` task again to check:
```dos
> gradlew.bat -q gdxVersion
1.9.5 (default: 1.9.8)
```
 
# Changelog

## 1.0.1
* Added `createAllTexturePacks` task which runs all texture pack tasks
* Added `createAllDistanceFields` task which runs all distance fields tasks
* Use `packSettings()` instead of `PackTextures.createSettings()` to create texture packer settings objects
* Made the plugin more Gradle Kotlin DSL-friendly. 