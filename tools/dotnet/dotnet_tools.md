
### Sample showing how to install nuget offline package locally
```
dotnet tool install --global --add-source C:\Users\Administrator\Downloads\dotnet-trace --ignore-failed-sources dotnet-trace
```

### See detailed outputs of dotnet tests
```
dotnet test --logger "console;verbosity=detailed"
```

### To run all the test in solution
```
dotnet test Path/To/YourSolution.sln
```
