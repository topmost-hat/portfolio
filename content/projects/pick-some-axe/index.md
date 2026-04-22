---
weight: -2

cover:
    image: "picksomeaxelogo.png"
    alt: "Pick Some Axe"
    caption: ""
    relative: false
---
Check out Pick Some Axe on: [Steam](https://store.steampowered.com/app/4459260/Pick_Some_Axe/), [itch.io](https://steveshack.itch.io/pick-some-axe)

### My Role: Gameplay / UI Programmer
* Made in Unity | Team Size: 13
* Refactored existing UI systems to make them more maintainable, and created new UI systems with a
focus on sustainability
* Designed new gameplay features for combat, puzzle-solving, and exploration
* Created developer tools with specific functions requested by other programmers and designers to
optimize the testing process of new features

atticus put the new trailer here once it exists

### Code Snippets
{{< collapse >}}
{{< collapse/above >}}
BreakableBarrel.cs
{{< /collapse/above >}}
{{< collapse/below >}}
``` c#
using System;
using UnityEngine;
using FMODUnity;

[Serializable]
public struct BarrelDrop
{
    public GameObject item;
    public float percentChance;
}

public class BreakableBarrel : MonoBehaviour
{
    [SerializeField] GameObject barrelBreakVFX;
    [SerializeField] BarrelDrop[] drops;
    [SerializeField] EventReference BreakSound;

    void Awake()
    {
        BreakSound = RuntimeManager.PathToEventReference("event:/SFX/Interactions/Barrel Break");
    }
    
    void OnTriggerEnter(Collider other)
    {
        // break when player hits this with pickaxe
        if(other.CompareTag("PickaxeHitbox")) { Break(); }
    }

    private void OnCollisionEnter(Collision collision)
    {
        // break when enemy is knocked back into this
        if (collision.collider.CompareTag("Enemy"))
        {
            EnemyScript enemy = collision.collider.GetComponent<EnemyScript>();

            // only break if enemy is actually in knockback
            if (enemy != null && enemy.stunned) { Break(); }
        }

        // break when player belly bashes this
        BellyDash bellyDash = collision.gameObject.GetComponent<BellyDash>();
        if (bellyDash != null && bellyDash.IsDashing) { Break(); }
    }

    public void Break()
    {
        DropItem();

        if(barrelBreakVFX) { Instantiate(barrelBreakVFX, transform.position, Quaternion.identity); }
        RuntimeManager.PlayOneShot(BreakSound, transform.position);

        // set persistence key so the barrel stays destroyed between play sessions
        PersistenceKey key = GetComponent<PersistenceKey>();
        if (null != key) { key.SetDestroyed(true); }

        Destroy(gameObject);
    }

    public void DropItem()
    {
        float rand = UnityEngine.Random.Range(0f, 100f);
        float accumulation = 0f;

        // loop through each item until the RNG number is within that item's "window"
        // accumulation creates windows by stacking each chance on top of each other
        // without having to define the ranges manually
        for(int i = 0; i < drops.Length; i++)
        {
            accumulation += drops[i].percentChance;
            if(rand <= accumulation)
            {
                if(null != drops[i].item) { Instantiate(drops[i].item, transform.position, Quaternion.identity); }
                i = drops.Length;
            }
        }
    }
}
```
{{< /collapse/below >}}
{{< /collapse >}}

{{< collapse >}}
{{< collapse/above >}}
GembitsUI.cs
{{< /collapse/above >}}
{{< collapse/below >}}
``` c#
using TMPro;
using UnityEngine;
using UnityEngine.UI;

public class GembitsUI : MonoBehaviour
{
    [SerializeField] float visibleTime = 3.0f;
    [SerializeField] float fadeOutTime = 1.0f;
    float visibleTimer = 0f;
    bool fadingOut = false;
    bool forceVisible = false;

    Image gembitsImage;
    TMP_Text gembitsText;

    void Awake()
    {
        gembitsImage = GetComponentInChildren<Image>();
        gembitsText = GetComponentInChildren<TMP_Text>();
    }

    void Start()
    {
        PlayerState.ListenToGembits(GembitsChanged);
        GembitsChanged(PlayerState.GetGembits());

        ShopHandler[] shops = FindObjectsOfType<ShopHandler>();
        foreach(ShopHandler shop in shops) { shop.ListenToInShop(InShop); }
    }

    void OnDestroy()
    {
        PlayerState.ListenToGembits(GembitsChanged, false);

        ShopHandler[] shops = FindObjectsOfType<ShopHandler>();
        foreach(ShopHandler shop in shops) { shop.ListenToInShop(InShop, false); }
    }

    void Update()
    {
        visibleTimer += Time.deltaTime;
        if(!forceVisible && !fadingOut && visibleTimer >= visibleTime)
        {
            fadingOut = true;
            gembitsImage.CrossFadeAlpha(0f, fadeOutTime, false);
            gembitsText.CrossFadeAlpha(0f, fadeOutTime, false);
        }
    }

    void ResetFadeOut()
    {
        visibleTimer = 0f;
        fadingOut = false;
        gembitsImage.CrossFadeAlpha(1f, 0f, true);
        gembitsText.CrossFadeAlpha(1f, 0f, true);
    }

    void GembitsChanged(int amount)
    {
        gembitsText.text = amount.ToString("D6");
        ResetFadeOut();
    }

    void InShop(bool inShop)
    {
        forceVisible = inShop;
        ResetFadeOut();
    }
}

```
{{< /collapse/below >}}
{{< /collapse >}}

{{< collapse >}}
{{< collapse/above >}}
LastSafePosition.cs
{{< /collapse/above >}}
{{< collapse/below >}}
``` c#
using System.Collections.Generic;
using UnityEngine;

[RequireComponent(typeof(ThirdPersonController))]
public class LastSafePosition : MonoBehaviour
{
    [SerializeField, Tooltip("Layers that count as safe ground.")] LayerMask groundLayers;
    [SerializeField] float raycastLength = 1.5f;

    // Minimum height for each level of safe positions
    [SerializeField] List<Transform> respawnFloors;
    float[] respawnFloorsY;
    Vector3[] safePositions;
    Vector3 overrideSafePosition = Vector3.positiveInfinity; // used during final climb
    bool safePosFound;

    readonly Vector3[] raycasts = {
        new(-1, -1, -1), new(0, -1, -1), new(1, -1, -1),
        new(-1, -1, 0),  new(0, -1, 0),  new(1, -1, 0),
        new(-1, -1, 1),  new(0, -1, 1),  new(1, -1, 1)
    };

    void Awake()
    {
        BubbleSortRespawnFloors();

        respawnFloorsY = new float[respawnFloors.Count];
        safePositions = new Vector3[respawnFloors.Count];
        for(int i = 0; i < respawnFloors.Count; i++)
        {
            respawnFloorsY[i] = respawnFloors[i].position.y;
            safePositions[i] = Vector3.positiveInfinity;
            Destroy(respawnFloors[i].gameObject);
        }
        respawnFloors.Clear();
    }

    void Start()
    {
        Vector3 returnPos = PlayerState.GetReturnPos();
        if(!Equals(Vector3.positiveInfinity, returnPos)
            && SceneState.GetScene() == PlayerState.GetReturnLevel())
        { GetComponent<ThirdPersonController>().Teleport(returnPos); }
    }

    void OnDestroy()
    {
        // try to set the return level
        ScenePSA prevScene = SceneState.GetPrevScene();
        bool success = PlayerState.SetReturnLevel(prevScene);

        if(ScenePSA.MAIN_MENU == prevScene) { return; } // special case: this is a purposeful failure
        else if(!success)
        {
            Debug.LogError("Failed to set return level to: " + SceneState.GetPrevScene());
            return;
        }

        // if that works, save last safe position as return position (if one exists)
        Vector3 returnPos;
        for(int i = 0; i < GetFloors(); i++)
        {
            returnPos = Get(transform.position.y, i);
            if(!Equals(Vector3.positiveInfinity, returnPos))
            {
                i = GetFloors();
                PlayerState.SetReturnPos(returnPos);
            }
        }

        PlayerState.Save(); // weird place to call this but it should be fine
    }

    void FixedUpdate()
    {
        safePosFound = true;

        // cast out a bunch of raycasts to determine what the ground is like around the player
        // if any of them return no ground collision, the position is not a valid respawn position
        for(int i = 0; i < 9; i++)
        {
            if(!Physics.Raycast(transform.position, raycasts[i], out _, raycastLength, groundLayers))
            {
                safePosFound = false;
                i = raycasts.Length;
            }
        }

        if(!safePosFound) { return; }

        // save safe position within highest possible floor
        for(int i = 0; i < safePositions.Length; i++)
        {
            if(transform.position.y > respawnFloorsY[i])
            {
                safePositions[i] = transform.position;
                //i = safePositions.Length;
            }
        }
    }

    // https://www.geeksforgeeks.org/dsa/bubble-sort-algorithm/
    void BubbleSortRespawnFloors()
    {
        int n = respawnFloors.Count;
        if(0 == n) { return; }

        bool swapped;
        for(int i = 0; i < n-1; i++) {
            swapped = false;
            for(int j = 0; j < n-i-1; j++)
            {
                // sort greatest to least
                if(respawnFloors[j].position.y < respawnFloors[j+1].position.y)
                {
                    (respawnFloors[j+1], respawnFloors[j]) = (respawnFloors[j], respawnFloors[j+1]);
                    swapped = true;
                }
            }

            if(!swapped) { i = n; }
        }
    }

    public Vector3 Get(float currentY, int backUp = 0)
    {
        // Safe position override has priority, used by final climb
        if(!Equals(Vector3.positiveInfinity, overrideSafePosition))
        {
            Vector3 temp = overrideSafePosition;
            overrideSafePosition = Vector3.positiveInfinity; // override decays after one respawn
            return temp;
        }

        // otherwise, determine which last safe position to use
        for(int i = 0; i < respawnFloorsY.Length; i++)
        {
            if(currentY >= respawnFloorsY[i])
            {
                if(backUp > i) { backUp = i; }
                return safePositions[i-backUp];
            }
        }

        Debug.LogWarning("Couldn't find a floor to respawn on!");
        return Vector3.positiveInfinity;
    }

    public int GetFloors() { return respawnFloorsY.Length; }

    public void SetOverrideSafePosition(Vector3 pos) { overrideSafePosition = pos; }
}

```
{{< /collapse/below >}}
{{< /collapse >}}
