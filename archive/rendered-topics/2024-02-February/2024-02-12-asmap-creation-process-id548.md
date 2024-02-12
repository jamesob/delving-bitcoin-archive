# ASMap creation process

fjahr | 2024-02-12 13:21:31 UTC | #1

(Cross-posting here for further discussion what I posted in the bitcoin-core-dev IRC meeting last week)

I want to share the progress we have made on the data collection and processing steps that can lead to an ASMap file that we trust enough to include it in releases.

[A rough draft of the ASMap creation process was shared about a year ago](https://gist.github.com/fjahr/f879769228f4f1c49b49d348f80d7635), now the tools have finally improved enough and the ideas of the process have matured that now we can show what it looks like in action.

The tools used are Kartograf (https://github.com/fjahr/kartograf) and sipa’s asmap-tool (https://github.com/bitcoin/bitcoin/pull/28793), the repository for the actual data is at https://github.com/fjahr/asmap-data for now, this should probably move into the org at some point if people are happy with this process.

Step 1: Data collection, demoed here: https://github.com/fjahr/asmap-data/issues/9 Multiple contributors first agree to launch the Kartograf mapping process at a specific timestamp, coordinated via an issue. Kartograf now has a feature that makes this comfortable (`-wait`), you just set it and it launches the process when the timestamp is hit. In testing so far we have found that there is a good chance that within a group of 5 or more the majority of participants will have the same result. That is the threshold I am proposing as well for the start: 5+ participants and >50% match in result. This will not work every time but maybe 4/5 and we can simply have a redo when there is a failure to agree on a result. We can also adjust the values over time as we get more practice. This process can be initiated by anyone, very similar to a Core PR. Participants that have a matching result could be interpreted as ACKs. If anyone sees something weird in the result or they simply didn’t get a match, they can ask for the raw data to be shared to investigate further.

Step 2: Compression, demoed here: https://github.com/fjahr/asmap-data/pull/10 Based on the result file a PR can be created that adds the compressed asmap.dat of the previous result to the repo. Similar to our normal PRs, 2-3 reviewers should ensure they get the same result before the file gets merged.

Step 3: Using the asmap.dat files: Users that want to run their node with an asmap file can get regular updates from there. When we decide to include a default asmap.dat file in a release, we can pick one of the recent ones from the repo.

I would be happy to receive further feedback on the process itself and the tooling. I will announce the next dates for creations here as well.

-------------------------

