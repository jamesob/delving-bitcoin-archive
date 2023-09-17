# Bitcoin developer tools suck, <possibly> Devimint fixes this

EthnTuttle | 2023-09-17 03:02:14 UTC | #1

https://bitcointv.com/w/gXuKzcErKxWDv3KKBHZSbA 
(Click bait title is click bait, but *possibly* correct)

What are your favorite developer tools for working within the bitcoin ecosystem?

I'll go first.

**Devimint** 
(see video for full nuanced discussion, see below for summary thoughts)

There is a Rust library/binary (it can be used as both) that has automated the standing up of local regtest bitcoin infrastructure for developer environments or CI/CD tests.

Let's look at a `match` arm of [`handle_command()`](https://github.com/fedimint/fedimint/blob/044236814450ad5ae83597f069147cc6d357e7ad/devimint/src/main.rs) because it is more elaborate that other commands I've documented and Fedimint uses it a lot in the tests. (The comment is from my local branch during exploration)
```
async fn handle_command() -> Result<()> {
    let args = Args::parse();
    match args.command { 
        ...
        // spins up bitcoind, cln w/ gateway, lnd w/ gateway, a faucet, electrs, esplora, and a
        // federation sized from FM_FED_SIZE it opens LN channel between the two nodes. it
        // connects the gateways to the federation. it finally switches to use the CLN
        // gateway using the fedimint-cli
        Cmd::DevFed => {
            let (process_mgr, task_group) = setup(args.common).await?;
            let main = async move {
                let dev_fed = dev_fed(&process_mgr).await?;
                dev_fed.fed.pegin(10_000).await?;
                dev_fed.fed.pegin_gateway(20_000, &dev_fed.gw_cln).await?;
                dev_fed.fed.pegin_gateway(20_000, &dev_fed.gw_lnd).await?;
                let daemons = write_ready_file(&process_mgr.globals, Ok(dev_fed)).await?;
                Ok::<_, anyhow::Error>(daemons)
            };
            cleanup_on_exit(main, task_group).await?;
        }
        ...
    Ok(())
}
```

Not only does it connect very commonly used bitcoin daemons/services, it then opens LN channels and generates blocks so it can pegin to a fully setup Fedimint. I've heard of but have not used a tool called Polar that seemingly does similar things, but with Docker. Fedimint couples this tool well with nix and mprocs.

Do please watch the click bait video. There is more nuanced ideas and discussion that I did not cover here. NixOS and nix-bitcoin is also mentioned.

So again, "What are your favorite developer tools for working within the bitcoin ecosystem?" and why is/isn't it devimint?

-------------------------

