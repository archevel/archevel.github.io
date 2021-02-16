---
layout: post
title:  "Adding single person sleep to multiplayer Valheim server"
date:   2021-02-16 13:00:00 +0000
categories: docker docker-compose haproxy 
---

Disclaimer: I give NO guarantees that the below doesn't mess up you Valheim game or characters!

Like many others I recently stared playing Valheim with a few friends. It is a nice co-op PvE game in a viking setting. However, one slight anoyance was the inability of triggering sleep unless ALL players on the server got into bed. So I decided to explore the code a bit using disassembly and see if I could modify the behaviour. I downloaded the Valheim dedicated server and installed [dnSpy](https://github.com/dnSpy/dnSpy). Using dnSpy I then opened the assembly `assembly_valheim.dll` which on my windows machine was found at `C:\Program Files (x86)\Steam\steamapps\common\Valheim dedicated server\valheim_server_Data\Managed` (this will depend a bit on where you decide to install the Valheim dedicated server).

In dnSpy's assembly explorer I could expand the dll file to find the different types definded. Scrolling down to `Game` and expanding it there is a private method `EverybodyIsTryingToSleep` which is the one we want to modify. It originally looked like:

```c#
// Game
// Token: 0x06000A80 RID: 2688 RVA: 0x0004C004 File Offset: 0x0004A204
private bool EverybodyIsTryingToSleep()
{
    List<ZDO> allCharacterZDOS = ZNet.instance.GetAllCharacterZDOS();
    if (allCharacterZDOS.Count == 0)
    {
        return false;
    }
    using (List<ZDO>.Enumerator enumerator = allCharacterZDOS.GetEnumerator())
    {
        while (enumerator.MoveNext())
        {
            if (!enumerator.Current.GetBool("inBed", false))
            {
                return false;
            }
        }
    }
    return true;
}
```

So modifying it to the below code changes the behaviour so that only a single player needs to go to a bed and try sleeping instead of all of them:

```c#
// Game
// Token: 0x06000A80 RID: 2688 RVA: 0x0004C004 File Offset: 0x0004A204
private bool EverybodyIsTryingToSleep()
{
    List<ZDO> allCharacterZDOS = ZNet.instance.GetAllCharacterZDOS();
    if (allCharacterZDOS.Count == 0)
    {
        return false;
    }
    using (List<ZDO>.Enumerator enumerator = allCharacterZDOS.GetEnumerator())
    {
        while (enumerator.MoveNext())
        {
            // This now checks if ANYONE is in bed
            if (enumerator.Current.GetBool("inBed", false))
            {
                return true;
            }
        }
    }
    // If no one is in bed we return false
    return false;
}
```

After the change all that was needed was for me to replace the assembly and start the server again. 

I tried this out with a friend on a server and it seem to work. Once someone goes to sleep the other person also is shown the sleep transition and "wakes up" the next day in the same spot they were. I also tried it out when I hadn't claimed a bed so that also seem to function as expected. Anyway I imagine that this should be added as a feature/toggle somewhere for servers, but I didn't want to do too much mucking about in the code base to add a startup flag to control the behaviour. This is just my hack to make Viking life a little more comfortable for me and my friends! And again just to be clear I can't make any claims if this is safe to do at all, but it was a fun evenings project!

Happy troll hunting!