# BLAST Patrol
The idea is to make sure all projects on GitHub know about the potential issues of using
`-max_target_seqs` parameter when running BLAST. The way we accomplish is to first search
for this particular, and rather unique, parameter name across all public repos on GitHub
and then create an issue on the repo with the code that uses this parameter to let the author
know about this.

## Search code
```python
from github import Github

g = Github(USERNAME, PERSONAL_ACCESS_TOKEN) # `repo` scope needed
hits = g.search_code('-max_target_seqs')
```

## Collect hits
```python

from time import sleep
import pandas as pd


# Collect hits
hits_list = []
for hit in hits:
    hits_list.append({'repo': hit.repository.full_name,
                   'file_url': hit.html_url,
                   'file_path': hit.path,
                   'file_name': hit.name})
    
    print('{}\t{}/{}'.format(len(hits_list), hit.repository.full_name, hit.path))
    sleep(2) # so we don't hit rate limits
```

## Create issues on corresponding repos with pointers to files

```python
# Turn them into a daraframe for better handling
hits_pd = pd.DataFrame.from_dict(hits_list)

issue_bdy = ("Hi there,\n"
             "\n"
             "This is a semi-automated message from a fellow bioinformatician. "
             "Through a GitHub search, I found that the following source files "
             "make use of BLAST's `-max_target_seqs` parameter: \n"
             "\n"
             "{}"
             "\n"
             "Based on the recently published report, "
             "[Misunderstood parameter of NCBI BLAST impacts the correctness of bioinformatics workflows]"
             "(https://academic.oup.com/bioinformatics/advance-article-abstract/doi/10.1093/bioinformatics/bty833/5106166?redirectedFrom=fulltext)"
             ", there is a strong chance that this parameter is misused in your repository.\n"
             "\n"
             "If the use of this parameter was intentional, please feel free to ignore and close this issue "
             "but I would highly recommend to add a comment to your source code to notify others about "
             "this use case. If this is a duplicate issue, please accept my apologies for the redundancy "
             "as this simple automation is not smart enough to identify such issues.\n"
             "\n"
             "Thank you!\n"
             "-- Arman ([armish/blast-patrol](https://github.com/armish/blast-patrol))")

repo_count = 0
for repo, hit_group in hits_by_repo:
    repo_count = repo_count + 1
    print("{}\t{}".format(repo_count, repo))
    
    try:
        gh_repo = g.get_repo(full_name_or_id=repo)

        files_str = ""
        for row, data in hit_group.iterrows():
            files_str = files_str + "- [{}]({})\n".format(data['file_path'], data['file_url'])

        gh_repo.create_issue("Confirm that use of BLAST's `-max_target_seqs` is intentional",
                             issue_bdy.format(files_str))
    except: # sometimes issues are disabled
        print('{} failed'.format(repo))
        
    sleep(5) # so we don't hit the limits
```
