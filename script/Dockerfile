FROM microsoft/dotnet:2.1-sdk AS build
WORKDIR /code
COPY . .
RUN dotnet restore
RUN dotnet publish --output /output --configuration Release

FROM microsoft/dotnet:2.1-aspnetcore-runtime
COPY --from=build /output /app
WORKDIR /app
ENTRYPOINT ["dotnet", "k8sdemo.dll"]