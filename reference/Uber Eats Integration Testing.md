---
tags:
---
# Uber Eats Integration Testing

Here are the methods and tools for testing the Uber Eats integration.

## Uber’s Sandbox Testing

-   **Documentation**: [Uber’s Sandbox Testing Doc](https://docs.google.com/document/d/17gmfZI2S2B1SKZHeDEEZZ1BSy3nuGTZxIhZVEkQkuG8/edit?tab=t.0)
-   **Notes from Brad**:
    -   [Slack Thread 1](https://cyan-bot.slack.com/archives/C07FHUAUHEH/p1746147661380039)
    -   [Slack Thread 2](https://cyan-bot.slack.com/archives/C089P6BUA56/p1749256075531489)
-   This should be the default testing strategy, and more time should be invested in improving it.

## Manual UE App Testing

1.  **Download the Uber Eats app**: This will not work with the web app.
2.  **Login with test credentials**:
    -   **Email**: `ashutosh.parashar+test+lucille+eater3@uber.com` (you can use eaters 1-3)
    -   **Password**: `Uber@12345`
3.  **Deploy Robot**: Deploy robot `C10190` to the [Uber QA Test Merchant](https://mission-control.delivery-stage.cocodelivery.com/v3/merchants/3dd84240-a8c5-11ed-a6d7-3b725b99f63b)’s Parking lot.
    -   The test merchant is **Yama Sushi Sake Attitude**.
    -   Only bots `C10190`, `C10661`, and `C10164` are onboarded with Uber. `C10190` should be used for Delivery Platform testing, but Uber may send the offer to any of the onboarded bots.
4.  **Log in as a pilot**.
5.  **Create an order in the app**: The fastest way is to re-order from a past order.

## Debugging tools

-   [Uber Integration Dashboard on DataDog](https://app.datadoghq.com/dashboard/myy-mmj-ksn/uber-integration?fromUser=false&refresh_mode=sliding&tpl_var_env%5B0%5D=staging&from_ts=1758478289382&to_ts=1758737489382&live=true)
-   [Uber Logs on DataDog](https://app.datadoghq.com/logs?query=env%3Astaging%20%40context%3A%2AUber%2A&agg_m=count&agg_m_source=base&agg_t=count&cols=host%2Cservice&messageDisplay=inline&refresh_mode=sliding&storage=hot&stream_sort=desc&viz=stream&from_ts=1758723136691&to_ts=1758737536691&live=true)
-   **Dispatch Visualizer Web App**: This is useful for understanding the supply and demand that the dispatch-engine sees. (Originally provided as `dispatch-visualizer.zip`)
