---
title: "Disable Security Manager"
date: 2019-04-13
draft: false
description: "Disabling the Java Security Manager"
tags: ["coding", "Java"]
---
# Introduction

Assume a security manager is present that doesn't grant the `setSecurityManager` permission. However, if it grants the following 3 permissions

- `accessDeclaredMembers`
- `suppressAccessChecks`
- `getProtectionDomain`

then the security manager can be disabled with the method we discuss in this article. Optionally, the permission `accessClassInPackage.jdk.internal.reflect` will make things more convenient.

# Problem
## `setSecurityManager` Is Guarded
The security manager guards certain methods from being invoked by callers without the appropriate permissions. Below is the source code of the `security` field, `getSecurityManager()` and `setSecurityManager(SecurityManager)` methods in `java.lang.System`.
```java
private static volatile SecurityManager security;

/**
 * Sets the System security.
 *
 * <p> If there is a security manager already installed, this method first
 * calls the security manager's <code>checkPermission</code> method
 * with a <code>RuntimePermission("setSecurityManager")</code>
 * permission to ensure it's ok to replace the existing
 * security manager.
 * This may result in throwing a <code>SecurityException</code>.
 *
 * <p> Otherwise, the argument is established as the current
 * security manager. If the argument is <code>null</code> and no
 * security manager has been established, then no action is taken and
 * the method simply returns.
 *
 * @param      s   the security manager.
 * @exception  SecurityException  if the security manager has already
 *             been set and its <code>checkPermission</code> method
 *             doesn't allow it to be replaced.
 * @see #getSecurityManager
 * @see SecurityManager#checkPermission
 * @see java.lang.RuntimePermission
 */
public static void setSecurityManager(final SecurityManager s) {
    if (security == null) {
        // ensure image reader is initialized
        Object.class.getResource("java/lang/ANY");
    }
    if (s != null) {
        try {
            s.checkPackageAccess("java.lang");
        } catch (Exception e) {
            // no-op
        }
    }
    setSecurityManager0(s);
}

private static synchronized
void setSecurityManager0(final SecurityManager s) {
    SecurityManager sm = getSecurityManager();
    if (sm != null) {
        // ask the currently installed security manager if we
        // can replace it.
        sm.checkPermission(new RuntimePermission
                                    ("setSecurityManager"));
    }

    if ((s != null) && (s.getClass().getClassLoader() != null)) {
        // New security manager class is not on bootstrap classpath.
        // Cause policy to get initialized before we install the new
        // security manager, in order to prevent infinite loops when
        // trying to initialize the policy (which usually involves
        // accessing some security and/or system properties, which in turn
        // calls the installed security manager's checkPermission method
        // which will loop infinitely if there is a non-system class
        // (in this case: the new security manager class) on the stack).
        AccessController.doPrivileged(new PrivilegedAction<>() {
            public Object run() {
                s.getClass().getProtectionDomain().implies
                    (SecurityConstants.ALL_PERMISSION);
                return null;
            }
        });
    }

    security = s;
}

/**
 * Gets the system security interface.
 *
 * @return  if a security manager has already been established for the
 *          current application, then that security manager is returned;
 *          otherwise, <code>null</code> is returned.
 * @see     #setSecurityManager
 */
public static SecurityManager getSecurityManager() {
    return security;
}
```
As you can see, calling `System.setSecurityManager(null)` will not work if a security manager is present and does not grant us the `setSecurityManager` permission.

## `security` Is Filtered from Reflection
Reflection is commonly used for accessing private class elements, such as private fields. However, the fields returned from the reflection methods go through a filter, removing those that are sensitive or unsafe. The `security` field in `java.lang.System` which holds a `java.lang.SecurityManager` is an example. Without the `Field` object, we cannot even do `AccessibleObject.setAccessible(boolean)`.

## Inspecting The Filter Class
Lets take a look of the source codes and track to the filter class. First, lets take a look at the method `getDeclaredFields` in `java.lang.Class` class.
```java
/**
 * Returns an array of {@code Field} objects reflecting all the fields
 * declared by the class or interface represented by this
 * {@code Class} object. This includes public, protected, default
 * (package) access, and private fields, but excludes inherited fields.
 *
 * <p> If this {@code Class} object represents a class or interface with no
 * declared fields, then this method returns an array of length 0.
 *
 * <p> If this {@code Class} object represents an array type, a primitive
 * type, or void, then this method returns an array of length 0.
 *
 * <p> The elements in the returned array are not sorted and are not in any
 * particular order.
 *
 * @return  the array of {@code Field} objects representing all the
 *          declared fields of this class
 * @throws  SecurityException
 *          If a security manager, <i>s</i>, is present and any of the
 *          following conditions is met:
 *
 *          <ul>
 *
 *          <li> the caller's class loader is not the same as the
 *          class loader of this class and invocation of
 *          {@link SecurityManager#checkPermission
 *          s.checkPermission} method with
 *          {@code RuntimePermission("accessDeclaredMembers")}
 *          denies access to the declared fields within this class
 *
 *          <li> the caller's class loader is not the same as or an
 *          ancestor of the class loader for the current class and
 *          invocation of {@link SecurityManager#checkPackageAccess
 *          s.checkPackageAccess()} denies access to the package
 *          of this class
 *
 *          </ul>
 *
 * @since 1.1
 * @jls 8.2 Class Members
 * @jls 8.3 Field Declarations
 */
@CallerSensitive
public Field[] getDeclaredFields() throws SecurityException {
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkMemberAccess(sm, Member.DECLARED, Reflection.getCallerClass(), true);
    }
    return copyFields(privateGetDeclaredFields(false));
}
```
The fields are obtained from `privateGetDeclaredFields(false)`.
```java
//
//
// java.lang.reflect.Field handling
//
//

// Returns an array of "root" fields. These Field objects must NOT
// be propagated to the outside world, but must instead be copied
// via ReflectionFactory.copyField.
private Field[] privateGetDeclaredFields(boolean publicOnly) {
    Field[] res;
    ReflectionData<T> rd = reflectionData();
    if (rd != null) {
        res = publicOnly ? rd.declaredPublicFields : rd.declaredFields;
        if (res != null) return res;
    }
    // No cached value available; request value from VM
    res = Reflection.filterFields(this, getDeclaredFields0(publicOnly));
    if (rd != null) {
        if (publicOnly) {
            rd.declaredPublicFields = res;
        } else {
            rd.declaredFields = res;
        }
    }
    return res;
}
```
As you can see, there is an invocation to the method `Reflection.filterFields`. That is the important line, where the fields get filtered. If you check the `import` statements, the `Reflection` class is actually `jdk.internal.reflect.Reflection`. Lets take a look at the filter method.
```java
public static Field[] filterFields(Class<?> containingClass,
                                    Field[] fields) {
    if (fieldFilterMap == null) {
        // Bootstrapping
        return fields;
    }
    return (Field[])filter(fields, fieldFilterMap.get(containingClass));
}
```
This method invokes a `filter` method, which takes an array of `Member`, and the names of the fields that should be filtered.
```java
private static Member[] filter(Member[] members, String[] filteredNames) {
    if ((filteredNames == null) || (members.length == 0)) {
        return members;
    }
    int numNewMembers = 0;
    for (Member member : members) {
        boolean shouldSkip = false;
        for (String filteredName : filteredNames) {
            if (member.getName() == filteredName) {
                shouldSkip = true;
                break;
            }
        }
        if (!shouldSkip) {
            ++numNewMembers;
        }
    }
    Member[] newMembers =
        (Member[])Array.newInstance(members[0].getClass(), numNewMembers);
    int destIdx = 0;
    for (Member member : members) {
        boolean shouldSkip = false;
        for (String filteredName : filteredNames) {
            if (member.getName() == filteredName) {
                shouldSkip = true;
                break;
            }
        }
        if (!shouldSkip) {
            newMembers[destIdx++] = member;
        }
    }
    return newMembers;
}
```
## Inspecting the Filter Maps
Finally, lets take at the declaration of the `fieldFilterMap`.
```java
/** Used to filter out fields and methods from certain classes from public
    view, where they are sensitive or they may contain VM-internal objects.
    These Maps are updated very rarely. Rather than synchronize on
    each access, we use copy-on-write */
private static volatile Map<Class<?>,String[]> fieldFilterMap;
private static volatile Map<Class<?>,String[]> methodFilterMap;

static {
    Map<Class<?>,String[]> map = new HashMap<Class<?>,String[]>();
    map.put(Reflection.class,
        new String[] {"fieldFilterMap", "methodFilterMap"});
    map.put(System.class, new String[] {"security"});
    map.put(Class.class, new String[] {"classLoader"});
    fieldFilterMap = map;

    methodFilterMap = new HashMap<>();
}
```
As you can see, the Javadoc also clearly states the functionality. By default, fields in `java.lang.System` named `security`, fields in `java.lang.Class` named `classLoader`, and fields in `jdk.internal.reflect.Reflection` named `fieldFilterMap` or `methodFilterMap`, are filtered. For methods, none are filtered by default.
# Solution: Obtain `security` Natively

## Get the `security` Field
If you look within the `privateGetDeclaredFields(boolean)` method, it actually calls the underlying native method `getDeclaredFields0(boolean)`, which returns a raw array of `Field`s. So, lets call the native method on `java.lang.Class<System>`, and select the `security` field.
```java
Method getDeclaredFields0M = Class.class.getDeclaredMethod("getDeclaredFields0", boolean.class);
getDeclaredFields0M.setAccessible(true);
Field[] fields = (Field[]) getDeclaredFields0M.invoke(System.class, false);
Field securityField = null;
for (Field field : fields) 
    if (field.getName().equals("security")) 
        securityField = field;
```
## Set Its Value via Reflection
Since we now have obtained the target field, what remains is to simply set the value via reflection.
```java
securityField.setAccessible(true);
securityField.set(null, null);
```
Thats it! The security field has been set to null. In other words, the Java Security Manager is disabled. You can check with `System.getSecurityManager() != null`, which will return `false`.

# Optional: Motifying the Filter Map
We can move one step further and remove `security` from `jdk.internal.reflect.Reflection.fieldFilterMap`. However, the `fieldFilterMap` also marks to filter itself, so we need to use solution 1 on `fieldFilerMap`. This solution also requires an additional permission of `accessClassInPackage.jdk.internal.reflect`, otherwise we are not allowed to access packages which are not exported from another module with reflection.
## Obtain `fieldFilterMap` Field
Since we cannot reference `jdk.internal.reflect.Reflection`, we need to use `Class.forName(String)`.
```java
Method getDeclaredFields0M = Class.class.getDeclaredMethod("getDeclaredFields0", boolean.class);
getDeclaredFields0M.setAccessible(true);
Field[] fields = (Field[]) getDeclaredFields0M.invoke(Class.forName("jdk.internal.reflect.Reflection"), false);
Field mapField = null;
for (Field field : fields) 
    if (field.getName().equals("fieldFilterMap")) 
        mapField = field;
```
## Get the Map via Reflection
We don't want to set the field to null, otherwise the filter will malfunction. Instead, get the `Map` and remove the entries.
### Inspect `setAccessible(boolean)`
However, `setAccessible(boolean)` will also explicitly check whether the Reflection class is exported to us, regardless of `accessClassInPackage` permission.
```java
/**
 * @throws InaccessibleObjectException {@inheritDoc}
 * @throws SecurityException {@inheritDoc}
 */
@Override
@CallerSensitive
public void setAccessible(boolean flag) {
    AccessibleObject.checkPermission();
    if (flag) checkCanSetAccessible(Reflection.getCallerClass());
    setAccessible0(flag);
}
```
The first suspicious check is `AccessibleObject.checkPermission()`, which routes to here.
```java
/**
 * The Permission object that is used to check whether a client
 * has sufficient privilege to defeat Java language access
 * control checks.
 */
private static final java.security.Permission ACCESS_PERMISSION =
    new ReflectPermission("suppressAccessChecks");

static void checkPermission() {
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) sm.checkPermission(ACCESS_PERMISSION);
}
```
This is not the cause. The second suspicious check is `checkCanSetAccessible(Reflection.getCallerClass())`, which routes to here.
```java
@Override
void checkCanSetAccessible(Class<?> caller) {
    checkCanSetAccessible(caller, clazz);
}
```
Which routes to `AccessibleObject.checkCanSetAccessible(Class<?>, Class<?>)`.
```java
void checkCanSetAccessible(Class<?> caller, Class<?> declaringClass) {
    checkCanSetAccessible(caller, declaringClass, true);
}
```
```java
private boolean checkCanSetAccessible(Class<?> caller,
                                        Class<?> declaringClass,
                                        boolean throwExceptionIfDenied) {
    Module callerModule = caller.getModule();
    Module declaringModule = declaringClass.getModule();

    if (callerModule == declaringModule) return true;
    if (callerModule == Object.class.getModule()) return true;
    if (!declaringModule.isNamed()) return true;

    String pn = declaringClass.getPackageName();
    int modifiers;
    if (this instanceof Executable) {
        modifiers = ((Executable) this).getModifiers();
    } else {
        modifiers = ((Field) this).getModifiers();
    }

    // class is public and package is exported to caller
    boolean isClassPublic = Modifier.isPublic(declaringClass.getModifiers());
    if (isClassPublic && declaringModule.isExported(pn, callerModule)) {
        // member is public
        if (Modifier.isPublic(modifiers)) {
            logIfExportedForIllegalAccess(caller, declaringClass);
            return true;
        }

        // member is protected-static
        if (Modifier.isProtected(modifiers)
            && Modifier.isStatic(modifiers)
            && isSubclassOf(caller, declaringClass)) {
            logIfExportedForIllegalAccess(caller, declaringClass);
            return true;
        }
    }

    // package is open to caller
    if (declaringModule.isOpen(pn, callerModule)) {
        logIfOpenedForIllegalAccess(caller, declaringClass);
        return true;
    }

    if (throwExceptionIfDenied) {
        // not accessible
        String msg = "Unable to make ";
        if (this instanceof Field)
            msg += "field ";
        msg += this + " accessible: " + declaringModule + " does not \"";
        if (isClassPublic && Modifier.isPublic(modifiers))
            msg += "exports";
        else
            msg += "opens";
        msg += " " + pn + "\" to " + callerModule;
        InaccessibleObjectException e = new InaccessibleObjectException(msg);
        if (printStackTraceWhenAccessFails()) {
            e.printStackTrace(System.err);
        }
        throw e;
    }
    return false;
}
```
As you can see, `InaccessibleObjectException` is thrown when `jdk.internal.reflect` is not exported.
### Call `setAccessible0(boolean)`
To solve this issue, we can simply invoke the `setAccessible0(boolean)` via reflection, which directly enables the suppressor for Java language access control.
```java
Method setAccessible0M = AccessibleObject.class.getDeclaredMethod("setAccessible0", boolean.class);
setAccessible0M.setAccessible(true);
setAccessible0M.invoke(mapField, true);
```
## Obtain the Map and Remove Entries
```java
((Map<?, ?>) mapField.get(null)).clear();
```
Generics are erased at runtime, so we can simply cast to `Map<?, ?>` and clear it or remove entries. If you want to modify the map by putting entries, you will need to cast to `Map<Class<?>, String[]>`, where the key is the class the fields are declared in, and the value an array storing the names of the fields to be filtered. Although there are utility methods in `Reflection` for you to mess with the filter configuration, the entire reflection process must be done again, which is annoying. Here is the source code of the utility methods.
```java
// fieldNames must contain only interned Strings
public static synchronized void registerFieldsToFilter(Class<?> containingClass,
                                            String ... fieldNames) {
    fieldFilterMap =
        registerFilter(fieldFilterMap, containingClass, fieldNames);
}

// methodNames must contain only interned Strings
public static synchronized void registerMethodsToFilter(Class<?> containingClass,
                                            String ... methodNames) {
    methodFilterMap =
        registerFilter(methodFilterMap, containingClass, methodNames);
}

private static Map<Class<?>,String[]> registerFilter(Map<Class<?>,String[]> map,
        Class<?> containingClass, String ... names) {
    if (map.get(containingClass) != null) {
        throw new IllegalArgumentException
                        ("Filter already registered: " + containingClass);
    }
    map = new HashMap<Class<?>,String[]>(map);
    map.put(containingClass, names);
    return map;
}
```
## Modify Cache If Any
If the above method did not work, it is most likely because the class you are getting `Member`s on has cached the `Member`s. If you look back on the `privateGetDeclaredFields(boolean)`, it actually obtains fields from `java.lang.Class.ReflectionData.declaredFields` field. Otherwise, it will obtain fields from `getDeclaredFields0()`, filter it, and cache it to `ReflectionData.declaredFields.` What this means is that the array of `Field` returned, which does not contain the filtered `Field`s, may actually be the old version before the filter got modified. Therefore, try to set the cache to the newer version.
```java
Method reflectionDataM = Class.class.getDeclaredMethod("reflectionData");
reflectionDataM.setAccessible(true);
Object reflectionData = reflectionDataM.invoke(System.class);
Field cacheField = reflectiondata.getClass().getDeclaredField("declaredFields");
cacheField.setAccessible(true);
cacheField.set(reflectionData, fields);
```