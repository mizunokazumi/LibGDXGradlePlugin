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
- [General](#general)
  - [LibGDX version](#libgdx-version)

# Getting started
Add the plugin to your project:

```groovy
plugins {
  id "com.github.blueboxware.gdx" version "1.0"
}
```

Create a packTextures task:

```groovy
packTextures {

  // The directory which contains the images to pack
  from 'textures/'
  
  // The target directory: 'pack.atlas' is placed in this directory
  into 'assets/'

  settings {
    // Settings for TexturePacker
    filterMin = "MipMapLinearNearest"
    filterMag = "MipMap"
  }
  
}
```

Run the task (or just do a build):

```dos
gradlew.bat packTextures
```

To create distance field tasks:

```groovy
distanceFields {

    // Creates a task called generateLogoDistanceField
    logo {
    
        inputFile = file('textures/logo.png')
        downscale = 8
        spread = 32
        outputFile = file('assets/logo-df.png')
        
    }
    
    // Creates a task called generateTitleDistanceField
    title {
    
        inputFile = file('textures/title.jpg')
        downscale = 4
        spread = 16
        color = 'ff0000'
        outputFile = file('assets/title-df.png')
    
    }

}
```

# PackTextures task

## Settings
Settings for Texture Packer are specified in a `settings { }` block. See [the LibGDX Wiki](https://github.com/libgdx/libgdx/wiki/Texture-packer#settings) 
for a list of available settings, their default values and descriptions. To get a quick overview of the available settings you can run the 
`texturePackerSettingsHelp` Gradle task.

For reference, these are the most important settings and their default values, as of LibGDX 1.9.8: 

```groovy
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
    filterMin = "Nearest"
    filterMag = "Nearest"
    wrapX = "ClampToEdge"
    wrapY = "ClampToEdge"
    format = "RGBA8888"
    alias = true
    outputFormat = "png"
    jpegQuality = 0.9
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
    scale = [1]
    scaleSuffix = [""]
    scaleResampling = ["bicubic"]
    atlasExtension = ".atlas"
    
}
```

## Generating multiple texture packs
If you want to create multiple texture packs, you can use `texturePacks { }` block.

The following example creates 3 tasks: pack**Game**Textures, pack**Menu**Textures and pack**GameOver**Textures:
```groovy
texturePacks {

    // Creates "game.atlas"
    game {
        from 'textures/game'
        into 'assets'
    }

    // Creates "menu.atlas"
    menu {
        from 'textures/menu'
        into 'assets'
        
        settings {
            filterMin = 'MipMapLinearNearest'
            filterMag = 'Nearest'
        }
    }

    gameOver {
        from 'textures/gameOver'
        into 'assets'
        // Name the pack "end.atlas" instead of the default "gameOver.atlas"
        packFileName = 'end.atlas'  
    }
    
}
```

## Reusing settings
To reuse settings for multiple texture packs, you can define settings objects with `PackTextures.createSettings { }`:

```groovy
import com.github.blueboxware.gdxplugin.tasks.PackTextures

// Create base settings
def packSettings = PackTextures.createSettings {
    filterMin = 'MipMapLinearNearest'
    filterMag = 'Nearest'
    maxWidth = 2048
    maxHeight = 2048
}

// Create settings for scaled texture packs based on the base settings
def scaledPackSettings = PackTextures.createSettings(packSettings) { 
    scale = [1, 2]
    scaleSuffix = ["Normal", "Scaled"]
    scaleResampling = ["bicubic", "bicubic"]
}

texturePacks {

    game {
        from 'textures/game'
        into 'assets'
        settings = packSettings
    }

    menu {
        from 'textures/menu'
        into 'assets'
        settings = scaledPackSettings
    }

    gameOver {
        from 'textures/gameOver'
        into 'assets'
        
        // Use packSettings, but change outputFormat to jpg
        settings = PackTextures.createSettings(packSettings) { 
            outputFormat = "jpg"
        }
    }
}
```

## Multiple input directories, filtering and renaming
Pack Textures tasks implement Gradle's [CopySpec](https://docs.gradle.org/current/javadoc/org/gradle/api/file/CopySpec.html), so you can specify
multiple input directories, and filter and rename files:

```groovy
packTextures {

    into 'assets'
    
    from('textures/ui') {
      exclude 'test*'
    } 
    
    from('textures/menu') {
      include '*.png'
      rename('menu_(.*)', '$1')
    }

}
```

## Using "pack.json"
Normally any `pack.json` files in the input directories (and any subdirectories) are ignored. If you want to load the texture packer settings from a 
pack.json file instead of defining them in build.gradle, you can use the `settingsFile` argument:

```groovy
packTextures {

  from 'textures/'
  into 'assets/'
  
  settingsFile = file('textures/pack.json')

}
```

If you want TexturePacker to use pack.json files found in the input directories and any subdirectories, set `usePackJson` to true:

```groovy
packTextures {

  from 'textures/'
  into 'assets/'
  
  usePackJson = true

}
``` 

Note that if you specify multiple input directories (see [above](#multiple-input-directories-filtering-and-renaming)), and more than one of the top level directories contain
pack.json files, only one of these is used. Use the `settingsFile` parameter to specify which one.

## Custom tasks
The plugin provides the `PackTextures` task type which can be used to create custom tasks:

```groovy
import com.github.blueboxware.gdxplugin.tasks.PackTextures

task('myPackTask', type: PackTextures) {

  description = 'Pack things'

  into 'assets'
  from 'textures'

  settings {
      atlasExtension = ".pack"
      filterMin = "MipMapLinearLinear"
  }
  
  doLast { 
      println 'Done!'
  }
  
}

// Run myPackTask on build
build.dependsOn(myPackTask)
```

Note that we added `myPackTask` to the dependencies of the `build` task so that myPackTask is automatically run when building the project. This is not necessary for the plugins builtin tasks (like `packTextures`):
they are automatically added to the build. 

# DistanceField task
The properties for the distance field task:

* `inputFile`: The input file (type: File)
* `outputFile`: The output file (type: File, default: inputFileWithoutExtension + "-df." + outputFormat) 
* `color`: The color of the output image (type: String, default: "ffffff")
* `downscale`: The downscale factor (type: int, default: 1)
* `spread`: The edge scan distance (type: float, default: 1.0)
* `outputFormat`: The output format (type: String, default: The extension of `outputFile`. If `outputFile` is not specified: "png") 

# General
## LibGDX version
The plugin comes with a bundled version of LibGDX Tools which is used for packing etc. To see the LibGDX version used by the plugin (this is not the version used by your project itself), run the `gdxVersion` task:

```dos
> gradlew.bat -q gdxVersion
1.9.8
```

If you want the plugin to use a different version, you can force this in the `buildscript` block:

```groovy
buildscript {

  repositories {
    mavenCentral()
    // ... other repositories
  }

  dependencies {
    // ... other dependencies
    classpath("com.badlogicgames.gdx:gdx-tools:1.9.5") {
      force = true
    }
  }
  
}
```

Use the `gdxVersion` task again to check:
```dos
> gradlew.bat -q gdxVersion
1.9.5 (default: 1.9.8)
```
 