# WordPress database clean up queries

## Orphan rows
Since WordPress uses MyISAM for it's storage engine, we don't get foreign keys - thus orphan rows can show themselves.

### wp_posts -> wp_posts (parent/child)

```sql
SELECT * FROM wp_posts
LEFT JOIN wp_posts child ON (wp_posts.post_parent = child.ID)
WHERE (wp_posts.post_parent <> 0) AND (child.ID IS NULL)

DELETE wp_posts FROM wp_posts
LEFT JOIN wp_posts child ON (wp_posts.post_parent = child.ID)
WHERE (wp_posts.post_parent <> 0) AND (child.ID IS NULL)
```

### wp_postmeta -> wp_posts

```sql
SELECT * FROM wp_postmeta
LEFT JOIN wp_posts ON (wp_postmeta.post_id = wp_posts.ID)
WHERE (wp_posts.ID IS NULL)

DELETE wp_postmeta FROM wp_postmeta
LEFT JOIN wp_posts ON (wp_postmeta.post_id = wp_posts.ID)
WHERE (wp_posts.ID IS NULL)
```

### wp_term_taxonomy -> wp_terms

```sql
SELECT * FROM wp_term_taxonomy
LEFT JOIN wp_terms ON (wp_term_taxonomy.term_id = wp_terms.term_id)
WHERE (wp_terms.term_id IS NULL)

DELETE wp_term_taxonomy FROM wp_term_taxonomy
LEFT JOIN wp_terms ON (wp_term_taxonomy.term_id = wp_terms.term_id)
WHERE (wp_terms.term_id IS NULL)
```

### wp_term_relationships -> wp_term_taxonomy

```sql
SELECT * FROM wp_term_relationships
LEFT JOIN wp_term_taxonomy ON (wp_term_relationships.term_taxonomy_id = wp_term_taxonomy.term_taxonomy_id)
WHERE (wp_term_taxonomy.term_taxonomy_id IS NULL)

DELETE wp_term_relationships FROM wp_term_relationships
LEFT JOIN wp_term_taxonomy ON (wp_term_relationships.term_taxonomy_id = wp_term_taxonomy.term_taxonomy_id)
WHERE (wp_term_taxonomy.term_taxonomy_id IS NULL)
```

### wp_usermeta -> wp_users

```sql
SELECT * FROM wp_usermeta
LEFT JOIN wp_users ON (wp_usermeta.user_id = wp_users.ID)
WHERE (wp_users.ID IS NULL)

DELETE wp_usermeta FROM wp_usermeta
LEFT JOIN wp_users ON (wp_usermeta.user_id = wp_users.ID)
WHERE (wp_users.ID IS NULL)
```

### wp_posts -> wp_users

```sql
SELECT * FROM wp_posts
LEFT JOIN wp_users ON (wp_posts.post_author = wp_users.ID)
WHERE (wp_users.ID IS NULL)

DELETE wp_posts FROM wp_posts
LEFT JOIN wp_users ON (wp_posts.post_author = wp_users.ID)
WHERE (wp_users.ID IS NULL)
```

## Other

### wp_postmeta dupes 
Checking for dupe `_wp_attached_file` / `_wp_attachment_metadata` keys (should only ever be one each per attachment post type).

```sql
SELECT post_id,meta_key,meta_value
FROM wp_postmeta
WHERE (meta_key IN('_wp_attached_file','_wp_attachment_metadata'))
GROUP BY post_id,meta_key
HAVING (COUNT(post_id) > 1)
```

### wp_postmeta dupes #2
Where an identical `meta_key` exists for the same post more than once.

```sql
SELECT *,COUNT(*) AS keycount
FROM wp_postmeta
GROUP BY post_id,meta_key
HAVING (COUNT(*) > 1)
```

### wp_postmeta missing 
Checking for missing `_wp_attached_file` / `_wp_attachment_metadata` keys on `wp_posts.post_type = 'attachment'` rows.

```sql
SELECT * FROM wp_posts
LEFT JOIN wp_postmeta ON (
	(wp_posts.ID = wp_postmeta.post_id) AND
	(wp_postmeta.meta_key = '_wp_attached_file')
)
WHERE (wp_posts.post_type = 'attachment') AND (wp_postmeta.meta_id IS NULL)

SELECT * FROM wp_posts
LEFT JOIN wp_postmeta ON (
	(wp_posts.ID = wp_postmeta.post_id) AND
	(wp_postmeta.meta_key = '_wp_attachment_metadata')
)
WHERE (wp_posts.post_type = 'attachment') AND (wp_postmeta.meta_id IS NULL)
```

### wp_options '_transient_' rows
A transient value is one stored by WordPress and/or a plugin generated from a complex query - basically a cache. More information on this can be found in this [answer on Stack Overflow](http://stackoverflow.com/a/11995022).

```sql
SELECT * FROM wp_options
WHERE option_name LIKE '%\_transient\_%'

DELETE FROM wp_options
WHERE option_name LIKE '%\_transient\_%'
```
