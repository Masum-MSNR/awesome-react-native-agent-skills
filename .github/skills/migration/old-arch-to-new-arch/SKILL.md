---
name: old-arch-to-new-arch
description: Guidance on migrating React Native apps from the Old Architecture (Paper + Bridge) to the New Architecture (Fabric + TurboModules) with Codegen and the interop layer. Use when asked about New Architecture migration, Fabric, TurboModules, bridgeless mode, or codegen.
---

# Migrating to the New Architecture (Fabric + TurboModules)

## Instructions

The New Architecture ships Fabric (new renderer), TurboModules (new native module system), Codegen, and bridgeless mode. Migrate incrementally using the interop layer.

### 1. Enable the Flags

`android/gradle.properties`:

```properties
newArchEnabled=true
hermesEnabled=true
```

`ios/Podfile`:

```ruby
ENV['RCT_NEW_ARCH_ENABLED'] = '1'
```

Reinstall pods:

```sh
cd ios && bundle exec pod install && cd ..
```

Then clean build both platforms. The interop layer transparently wraps non-migrated legacy native modules and view managers, so you can enable the flag before every dependency is fully New-Arch ready.

### 2. Audit Dependencies

Run this quick audit before flipping flags:

```sh
pnpm why react-native
# Inspect each native dep's package.json for `codegenConfig` or "newArchEnabled" support.
```

For each native dep, check:

- Does it ship `codegenConfig` in `package.json`?
- Does it declare Fabric view managers (`*ComponentView`)?
- Does its `build.gradle` / `podspec` compile with `newArchEnabled=true`?

Deps without New-Arch support usually still work via the interop layer, but should be upgraded or replaced.

### 3. Codegen Configuration

Add a `codegenConfig` block to your app's `package.json`:

```json
{
  "codegenConfig": {
    "name": "AppSpec",
    "type": "all",
    "jsSrcsDir": "src/specs",
    "android": { "javaPackageName": "com.example.app.spec" }
  }
}
```

### 4. A TurboModule Spec

`src/specs/NativeMathSpec.ts`:

```ts
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  add(a: number, b: number): number;
  hashAsync(value: string): Promise<string>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('NativeMath');
```

Consume it:

```ts
import NativeMath from '@specs/NativeMathSpec';

const sum = NativeMath.add(1, 2);
const hash = await NativeMath.hashAsync('hello');
```

Codegen will emit the required C++/Java/Obj-C++ bindings during the build. You only need to implement the native side.

### 5. A Fabric View Component

`src/specs/RNCustomMapNativeComponent.ts`:

```ts
import type { ViewProps } from 'react-native';
import type { HostComponent } from 'react-native';
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';
import type { Double } from 'react-native/Libraries/Types/CodegenTypes';

export interface NativeProps extends ViewProps {
  zoom?: Double;
  showUser?: boolean;
}

export default codegenNativeComponent<NativeProps>('RNCustomMap') as HostComponent<NativeProps>;
```

### 6. Bridgeless Mode

Bridgeless mode removes the legacy JS-to-native bridge entirely. Turn it on once TurboModules are healthy:

```properties
# android/gradle.properties
react.bridgelessDisabled=false
```

```ruby
# ios/Podfile
use_react_native!(:bridgeless_enabled => true)
```

### 7. Verification

Detect at runtime to assert migration is complete in CI smoke tests:

```ts
export const newArch = {
  fabric: global?.nativeFabricUIManager != null,
  turbo: (global as { RN$Bridgeless?: boolean }).RN$Bridgeless === true,
};
```

Gate releases on `fabric === true` before removing the old architecture escape hatch.

### 8. Common Migration Issues

- **Direct UIManager calls**: `UIManager.dispatchViewManagerCommand` is replaced by ref-based imperative handles on Fabric components. Fix any legacy calls.
- **`setNativeProps`**: prefer state or shared values (Reanimated). `setNativeProps` works via interop but is deprecated.
- **Event dispatchers**: custom view managers must expose events via Codegen `bubblingEventTypes`/`directEventTypes` in the component spec.
- **`measure`/`measureInWindow`**: still work but go through Fabric's commit phase, so results may be delayed by one frame.

## Checklist

- [ ] `newArchEnabled=true` on Android and `RCT_NEW_ARCH_ENABLED=1` on iOS.
- [ ] App's `package.json` declares `codegenConfig` and specs live under a dedicated folder.
- [ ] Every new native module is a TurboModule spec; every new view is a codegen Fabric component.
- [ ] Dependencies without New-Arch support are tracked in a migration issue and work via interop only temporarily.
- [ ] Bridgeless mode is enabled in CI and verified at runtime.
- [ ] No `UIManager.dispatchViewManagerCommand` or `setNativeProps` calls in new code.
