You don’t need to guess it — it’s simply the number of rows in your “auto\_approved” subset of the live data.  As soon as you do:

```python
prod_auto = get_prod_data_updated("2024-07-01","2024-12-31", subset="auto_approved")
```

you can just inspect

```python
auto_volume = prod_auto.shape[0]
print("Auto-approved volume:", auto_volume)
```

If instead you have already built the set of approved ROWIDs:

```python
auto_approved_rowids = set(...)   # from your step [16]
print("Auto-approved volume:", len(auto_approved_rowids))
```

Either one will give you the precise count.  In fact, in your notebook you saw:

```
num auto approved appids 599252
```

That is the volume of truly auto-approved applications over your scoring window.
